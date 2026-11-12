# BCC Grid Rendering

Also called: Body-Centered Cartesian Grid Rendering, BCC Splatting

## Overview
BCC Grid Rendering utilizes a Body-Centered Cartesian (BCC) lattice to store and render volumetric data, reducing the required number of sample points to 70.3% of a standard Cubic Cartesian (CC) grid while maintaining lossless compression for spherically bandlimited signals [S1, unofficial][S2, unofficial]. It is distinct from CC-based splatting by its specific lattice geometry and the requirement to shift rendering slices by half a unit, making it optimal for point-based object-order methods that employ spherical filters [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset stored on a BCC lattice, defined by sample positions $\mathbf{p}_i$, scalar values (e.g., density $\rho_i$), and a view matrix.
*   **Output:** A 2D raster image.

**Data Structures:**
The BCC grid is a subset of the 3D integer lattice $\mathbb{Z}^3$. It consists of two interleaved CC grids. If the base CC grid has spacing $h$, the BCC grid samples are located at:
1.  Integer coordinates $(i, j, k)$ where $i+j+k$ is even.
2.  Half-integer coordinates $(i+0.5, j+0.5, k+0.5)$ where $i+j+k$ is even.
Alternatively, it can be viewed as a CC grid with spacing $h$ plus a CC grid with spacing $h$ shifted by $(0.5, 0.5, 0.5)h$. The total sample density is $\frac{2}{\sqrt{2}} \approx 0.707$ times that of a CC grid with the same nearest-neighbor distance, or 70.3% of a CC grid with the same unit cell volume [S1, unofficial][S2, unofficial].

**Steps:**
1.  **Lattice Transformation:** Transform the BCC sample positions from volume space to screen space using the Model-View-Projection (MVP) matrix, consistent with standard splatting [S1, unofficial].
2.  **Slice Shifting:** When rendering via slice-based or ordered accumulation, shift the BCC slices by $1/2$ relative to the CC grid alignment to correctly interleave the two sub-lattices [S1, unofficial].
3.  **Splatting:** Project each sample point onto the image plane. Use a spherical (radially symmetric) reconstruction kernel $h(\mathbf{r})$ [S2, unofficial].
4.  **Compositing:** Accumulate the splatted contributions using alpha blending [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume samples is a list of (x, y, z, value) in BCC lattice coordinates
# Assume kernel is a spherical function h(r)
# Assume view_matrix transforms volume to clip space

def render_bcc(samples, view_matrix, image_width, image_height):
    image = zeros((image_height, image_width))
    
    # 1. Transform and Project
    splats = []
    for (x, y, z, val) in samples:
        # Transform to clip space
        clip_pos = view_matrix * [x, y, z, 1]
        
        # Perspective divide
        if clip_pos.w <= 0:
            continue
        ndc_x = clip_pos.x / clip_pos.w
        ndc_y = clip_pos.y / clip_pos.w
        
        # Map to screen coordinates
        screen_x = (ndc_x + 1) * 0.5 * image_width
        screen_y = (1 - (ndc_y + 1) * 0.5) * image_height # Y is inverted in screen coords
        
        # 2. Shift slices by 1/2 (Conceptual: The lattice itself is shifted, 
        # or the rendering order/offset accounts for the BCC structure)
        # In practice, this is handled by the lattice definition in step 1.
        
        splats.append({
            'x': screen_x,
            'y': screen_y,
            'value': val,
            'weight': 1.0 # Standard splat weight
        })
        
    # 3. Rasterize Splats
    for splat in splats:
        # Determine footprint radius based on kernel support
        radius = get_kernel_radius() 
        
        # Iterate over bounding box of the splat
        x_min = int(splat['x'] - radius)
        x_max = int(splat['x'] + radius)
        y_min = int(splat['y'] - radius)
        y_max = int(splat['y'] + radius)
        
        for sx in range(x_min, x_max):
            for sy in range(y_min, y_max):
                # Calculate distance from center
                dx = sx - splat['x']
                dy = sy - splat['y']
                dist_sq = dx*dx + dy*dy
                
                # Evaluate spherical kernel
                # Note: Trilinear filters should NOT be used here [S2, unofficial]
                weight = evaluate_spherical_kernel(dist_sq)
                
                # Accumulate
                image[sy, sx] += splat['value'] * weight * splat['weight']
                
    return image
```

**Parameters:**
*   **Kernel:** Must be radially symmetric (e.g., Gaussian) to preserve the spectral properties of the BCC lattice [S2, unofficial].
*   **Slice Shift:** $0.5$ units in the relevant axis if using slice-based traversal [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Aliasing:** Using trilinear interpolation filters on a BCC grid leads to aliasing because the trilinear filter's frequency response is not radially symmetric, failing to suppress higher spectra correctly [S2, unofficial].
*   **Dimensionality:** The 50% reduction claim applies specifically to 4D (time-varying) BCC grids compared to 4D CC grids [S1, unofficial].

## Subtopics

### BCC Grid Sampling
BCC Grid Sampling refers to the storage and acquisition of volume data on a Body-Centered Cartesian lattice, which is the dual of the Face-Centered Cartesian (FCC) lattice [S2, unofficial]. It is chosen when the signal is spherically bandlimited, allowing for fewer samples than a Cubic Cartesian (CC) grid without loss of information [S2, unofficial]. It is best suited for point-based object-order methods using spherical filters [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering a BCC grid takes approximately 70% of the time required for a CC grid rendering the same visual quality [S2, unofficial]. For 4D datasets, BCC reduces the sample count to 50% of the equivalent CC grid [S1, unofficial].
*   **Memory Complexity:** Requires 70.3% of the memory of a CC grid for 3D data [S1, unofficial].
*   **Performance:** In the "Foot" dataset, BCC rendering took 70% of the time of CC rendering [S2, unofficial]. In the "Turbulent Jet" 4D dataset, BCC rendered in 1.01s (71% of CC time) with 138 time steps, compared to 99 time steps for CC [S1, unofficial].
*   **Strengths:** Significant reduction in sample count and memory usage for spherically bandlimited signals; lossless compression under that assumption [S1, unofficial].
*   **Limitations:** Requires spherical filters; incompatible with standard trilinear interpolation without aliasing artifacts [S2, unofficial]. Not as common as CC grids in general-purpose software [S2, unofficial].

## Background
The BCC grid was introduced as an alternative to the Cubic Cartesian (CC) grid for volumetric rendering to improve efficiency. Theussl (2001) noted that BCC grids allow the use of all usual point rendering methods with a simple slice shift [S1, unofficial]. Neophytou (2002) demonstrated the performance benefits on 4D turbulent jet data [S1, unofficial]. Chen et al. and Yagel and Shi are cited in the context of related acceleration techniques, but the specific BCC lattice properties are attributed to the Fourier analysis of space-filling points [S2, unofficial].

## Sources
[S1] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
