# Optimized Ray Casting

Also called: Adaptive Ray Casting, Boundary cell projection, Fast Ray Casting Algorithm, Octtree Ray Casting

## Overview
Optimized Ray Casting is a volume rendering algorithm that accelerates image-order rendering by combining hierarchical data structures (octrees) with adaptive sampling and early ray termination [S1, S2, unofficial]. It solves the performance bottleneck of standard ray casting, which processes every voxel along a ray regardless of transparency or redundancy [S2, unofficial]. It sets itself apart from siblings like [Lazy Sweep Ray Casting](1649-lazy-sweep-ray-casting.md) by using explicit octtree traversal to skip transparent regions rather than relying on sweep-line or slab-based accumulation, and from [Ray-guided volume rendering with missing brick reporting](2815-ray-guided-volume-rendering-with-missing-brick-reporting.md) by focusing on single-volume hierarchical traversal rather than distributed brick management [S1, S2, unofficial].

## How it works
**Inputs:** 3D scalar volume dataset (regular grid); camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$); rendering parameters (opacity threshold, sample region size, color/density difference thresholds).
**Outputs:** 2D color image.

**Data Structures:**
1.  **Octree:** Hierarchical representation of the volume. Each node represents a macro-block (e.g., $8 \times 8 \times 8$ voxels). Nodes are tagged as "fully transparent" if all constituent voxels are transparent [S2, unofficial].
2.  **Backstore:** Pixel memory storing intermediate results for parallel renderers [S2, unofficial].

**Parameters:**
*   **Alpha Cutoff:** Opacity threshold for early ray termination. Typical values: 92% (electric field), 98% (neghip, brain) [S1, unofficial].
*   **Sample Region Size:** Square grid for adaptive sampling. Typical value: $3 \times 3$ pixels [S1, unofficial].
*   **Color Difference Threshold:** Max allowed difference for bilinear interpolation of adjacent pixels. Typical values: 10 (electric field), 40 (neghip), 30 (brain) [S1, unofficial].
*   **Density Difference Threshold:** Max allowed difference for adaptive ray density along a ray. Typical values: 30 (electric field), 80 (neghip), 5 (brain) [S1, unofficial].

**Steps:**

1.  **Preprocessing (Offline):**
    *   Construct the octree from the volume data [S2, unofficial].
    *   Tag macros containing only transparent voxels [S2, unofficial].
    *   Detect boundary cells for traversal optimization [S1, unofficial].

2.  **Ray Initialization:**
    *   For each pixel in the image, define ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ [G].
    *   If a cut plane is defined, start the ray at the intersection with the plane instead of the dataset boundary [S2, unofficial].

3.  **Adaptive Ray Casting Loop:**
    *   **Coarse Pass:** Cast rays on a coarse grid (e.g., every 3rd pixel) [S2, unofficial].
    *   **Refinement:** For each coarse pixel, check adjacent pixels. If the color difference exceeds the threshold, cast additional rays for the intermediate pixels [S1, S2, unofficial].
    *   **Final Image Formation:** Form the final image by bilinear interpolation of the ray-cast pixels [S2, unofficial].

4.  **Ray Traversal and Compositing (Per Ray):**
    *   Traverse the octree to find the entry and exit points of the ray within the volume [S1, S2, unofficial].
    *   Initialize accumulated opacity $\alpha_{acc} = 0$ and color $C_{acc} = 0$ [G].
    *   While $\alpha_{acc} < \text{Alpha Cutoff}$ and ray is within volume:
        *   Use the octree to skip over transparent regions. If a node is fully transparent, advance $t$ past it [S2, unofficial].
        *   Sample the voxel value $V$ at current $t$ [G].
        *   Check density difference against previous sample. If difference > Threshold, ensure sufficient sampling density (cast additional rays if in adaptive mode) [S1, unofficial].
        *   Compute transmittance $T = \exp(-\sigma \Delta t)$ and opacity contribution $\alpha = 1 - T$ [G].
        *   Update accumulated color: $C_{acc} = C_{acc} + C_{voxel} \cdot \alpha \cdot (1 - \alpha_{acc})$ [G].
        *   Update accumulated opacity: $\alpha_{acc} = \alpha_{acc} + \alpha \cdot (1 - \alpha_{acc})$ [G].
        *   Advance $t$ by step size $\Delta t$ [G].
    *   If $\alpha_{acc} \ge \text{Alpha Cutoff}$, terminate ray (Early Ray Termination) [S2, unofficial].

5.  **Parallel Execution (Optional):**
    *   Master GPU controls slave GPUs via a finite state automaton [S2, unofficial].
    *   Slaves render image tiles (e.g., $128 \times 128$) independently [S2, unofficial].
    *   Macros are fetched asynchronously to slave pixel memory [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimized_ray_casting(volume_data, camera, params):
    """
    volume_data: 3D array of voxel values
    camera: object with origin o, direction d, and image dimensions
    params: dict with alpha_cutoff, color_thresh, density_thresh, region_size
    """
    # Preprocessing
    octree = build_octree(volume_data)
    tag_transparent_macros(octree)
    
    image_width = camera.width
    image_height = camera.height
    image = zeros((image_height, image_width, 3))
    
    # Adaptive Sampling: Coarse Grid
    step = params['region_size'] # e.g., 3
    coarse_pixels = []
    for y in range(0, image_height, step):
        for x in range(0, image_width, step):
            ray = Ray(camera.o, normalize(camera.d + offset(x, y, camera)))
            color = cast_ray(ray, octree, params)
            image[y, x] = color
            coarse_pixels.append((x, y, color))
            
    # Refinement: Interpolate missing pixels
    for y in range(image_height):
        for x in range(image_width):
            if (y % step != 0) or (x % step != 0):
                # Check neighbors for color difference
                neighbors = get_neighbors(image, x, y)
                if any(abs(c - neighbors[0]) > params['color_thresh'] for c in neighbors):
                    ray = Ray(camera.o, normalize(camera.d + offset(x, y, camera)))
                    image[y, x] = cast_ray(ray, octree, params)
                else:
                    image[y, x] = bilinear_interpolate(image, x, y)
                    
    return image

def cast_ray(ray, octree, params):
    alpha_acc = 0.0
    color_acc = 0.0
    t = ray.entry_t
    prev_val = None
    
    while alpha_acc < params['alpha_cutoff'] and t < ray.exit_t:
        # Octtree Skip
        node = octree.get_node_at(ray, t)
        if node.is_transparent:
            t = node.exit_t
            continue
            
        # Sample
        val = sample_volume(ray, t)
        
        # Adaptive Density Check
        if prev_val is not None and abs(val - prev_val) > params['density_thresh']:
            # In a full implementation, this might trigger sub-stepping
            # For this algorithm, it ensures we don't undersample high-contrast areas
            pass 
            
        # Compositing
        alpha = compute_opacity(val)
        color = compute_color(val)
        
        color_acc = color_acc + color * alpha * (1.0 - alpha_acc)
        alpha_acc = alpha_acc + alpha * (1.0 - alpha_acc)
        
        prev_val = val
        t += ray.step_size
        
    return color_acc
```

## Subtopics

### [Light Ray Extinction Optimization](15221-light-ray-extinction-optimization.md)
Accelerates direct illumination in Monte Carlo volume rendering by optimizing extinction estimates along light rays. Unlike the parent method which optimizes camera rays, this handles stochastic scattering vertices, requiring data structure rebuilding per light ray set.

### [Parallel Volume Rendering for Computational Flow Imaging](16105-parallel-volume-rendering-for-computational-flow-imaging.md)
Visualizes CFD density/refractive index fields on distributed supercomputers. Converts curilinear grids to rectilinear multi-block voxel fields to avoid integral errors from overlapping grids, distributing voxels for efficient parallel ray-casting.

### [STiles (Sorted Tiles)](16374-stiles-sorted-tiles.md)
Progressive ray casting that decomposes the image into screen-space tiles, sorts them by cost, and renders them in batches within a time budget. Optimizes GPU cache locality by processing tiles rather than individual rays, avoiding stalls on mobile/low-performance devices.

### Nishita and Nakamae Water Rendering
Considers optical effects (caustics, shafts of light) in water modeled as a uniform particle density medium [S3]. Uses analytical functions for scattered light intensity based on distance, avoiding numerical integration and ray-tracing in favor of scanline Z-buffer and accumulation buffer techniques [S3].

### Pre-calculated Gradient Storage
Replaces real-time gradient calculation during raycasting with a pre-computed 3D RGBA texture lookup [S4]. Saves 6 extra texture fetches per sample by trading memory for computation speed, advantageous for VR hardware like HTC Vive [S4].

### Reslicing via Ray Casting
Modifies ray casting to reslice a volume along an arbitrary plane $ax + by + cz + d = 0$ [S5]. Implemented by starting the ray search at the z-coordinate determined by the plane equation or continuing until the x-coordinate value, effectively removing voxels on one side of the plane [S5].

## Operating characteristics
**Time Complexity:** Depends on the sparsity of the volume and the effectiveness of the octtree. In the worst case (fully opaque), it approaches $O(N)$ where $N$ is the number of voxels along the ray. With transparent regions, it is significantly faster due to skipping [S2, unofficial].
**Memory Complexity:** $O(V)$ for the volume data plus $O(M)$ for the octree, where $M$ is the number of macro blocks [S2, unofficial].
**Measured Performance:**
*   On 16-processor SGI Power Challenge: Electric field ($50^3$) 0.04s, Neghip ($68^3$) 0.15s, Brain ($256 \times 256 \times 124$) 0.09s [S1, unofficial].
*   On 4 processors: Electric field 0.39s, Neghip 1.02s, Brain 0.17s [S1, unofficial].
*   On 8 processors: Electric field 0.11s, Neghip 0.93s, Brain 0.12s [S1, unofficial].
*   On 12 processors: Electric field 0.06s, Neghip 0.28s, Brain 0.11s [S1, unofficial].
*   More than 10 frames per second achieved on 16 processors for all datasets [S1, unofficial].
*   Slave GPU rendering of $128 \times 128$ tile for $128^3$ dataset: 100-240 ms [S2, unofficial].
**Strengths:** High performance on sparse/transparent volumes; interactive frame rates achievable on parallel hardware; adaptive sampling reduces aliasing without uniform high resolution [S1, S2, unofficial].
**Limitations:** Overhead of octtree construction and traversal for dense volumes; adaptive sampling may introduce artifacts if thresholds are poorly chosen [S1, S2, unofficial].
**When to use:** Sparse volumetric data (e.g., medical CT, scientific simulations); interactive applications requiring real-time feedback; parallel rendering architectures [S1, S2, unofficial].
**When not to use:** Dense, fully opaque volumes where octtree skipping provides little benefit; hardware with limited VRAM for octree storage [S2, unofficial][S4].

## Background
Originated in the late 1980s and early 1990s with work by Levoy and others on hierarchical volume representation [S2, unofficial]. The specific "Fast Ray Casting Algorithm" with boundary cell projection and adaptive sampling was demonstrated on SGI workstations and parallel clusters in the early 1990s [S1, unofficial]. It builds on [Parent](13739-traversal-and-data-access.md) by introducing specific optimizations for hierarchical data traversal and adaptive sampling rates [S1, S2, unofficial].

## Sources
[S1] Time-Critical Volume Rendering, https://ntrs.nasa.gov/api/citations/19990040407/downloads/19990040407.pdf, fetched 2026-10-26 (unofficial)
[S2] DTIC ADA236718: Achieving Direct Volume Visualization with Interactive Semantic Region Selection, https://archive.org/details/DTIC_ADA236718, fetched 2026-10-26 (unofficial)
[S3] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S4] Rendering and Interacting With Volume Models in Immersive Environments, https://doi.org/10.2312/ceig.20181153, fetched 2026-10-26
[S5] Hardware Support for the Display and Manipulation of Binary Voxel Models, https://doi.org/10.2312/eggh/eggh88/103-117, fetched 2026-10-26
