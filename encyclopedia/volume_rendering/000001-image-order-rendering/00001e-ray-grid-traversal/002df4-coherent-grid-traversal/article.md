# Coherent Grid Traversal

Also called: CGT, CGT algorithm, Ize et al. traversal

## Overview
Coherent Grid Traversal (CGT) is a ray-grid traversal method that accelerates ray tracing in uniform grids by processing bundles of coherent rays as frustums, advancing slice-by-slice along a dominant axis rather than cell-by-cell [S4, S5]. It solves the packet divergence problem inherent in cell-by-cell traversal (like 3DDDA), where rays in a group disagree on the next cell, by maintaining packet coherence through incremental frustum-slice overlap computation [S2, S4]. Unlike hierarchical structures like kd-trees, CGT uses a flat uniform grid, enabling extremely fast per-frame rebuilds suitable for fully dynamic scenes [S3, unofficial][S4].

## How it works
**Inputs and Outputs:**
*   **Input:** A packet of $N \times N$ coherent rays (primary or secondary), a uniform grid of dimensions $N_x \times N_y \times N_z$, and the grid's bounding box. For volume rendering, the input is a "bricked" volume [S1, S8, unofficial].
*   **Output:** A sequence of grid cells (or bricks) overlapped by the ray packet frustum, along with the ray-cell intersection intervals.

**Data Structures:**
*   **Canonical Grid Space:** Rays are transformed into a coordinate system where the grid maps to $[0, N_x) \times [0, N_y) \times [0, N_z)$ [S4, S5]. Cell coordinates are obtained by truncating floating-point positions to integers [S4, S5].
*   **Frustum Representation:** Defined by four bounding planes in the minor axes ($\tilde{U}, \tilde{V}$) and two planes in the major axis ($\tilde{K}$) [S4, S6]. For primary rays, the four corner rays define these planes [S2, S4].
*   **Overlap Box $B(i)$:** A 2D bounding box in the $\tilde{U}, \tilde{V}$ plane representing the frustum's intersection with slice $i$ along the major axis $\tilde{K}$ [S4, S5]. It is stored as four floats: $[u_{min}, v_{min}, u_{max}, v_{max}]$ [S9, unofficial].
*   **Bricked Volume (Volume Rendering):** The volume is stored as bricks referenced by 3 maps (one per principal axis). Empty bricks are not stored to achieve compression [S1, S8, unofficial].

**Parameters:**
*   **Packet Size:** Typically $4 \times 4$ or $8 \times 8$ rays [S6, S7].
*   **Coherence Constraint:** Rays in a packet must span an angle of less than $\pi/2$ to ensure they share the same sign along the major traversal axis [S2, S4, S5].
*   **Macrocell Size (Hierarchical):** $M \times M \times M$ blocks of fine cells (e.g., $6 \times 6 \times 6$) [S7, S4].

**Algorithm Steps:**

1.  **Transform to Canonical Space:** Transform all rays in the packet into the canonical grid coordinate system [S4, S5].
2.  **Determine Major Axis:** Select the major traversal axis $\tilde{K}$ based on the dominant component ($\pm X, \pm Y, \pm Z$) of the direction of the first ray in the packet [S4, S5]. The remaining axes are $\tilde{U}$ and $\tilde{V}$ [S4, S5].
3.  **Compute Frustum Planes:** Calculate the four bounding planes of the frustum in $\tilde{U}$ and $\tilde{V}$ using the extremal slopes of the rays along $\tilde{K}$ [S4, S5]. For primary packets, use the four corner rays [S4].
4.  **Clip to Grid:** Intersect the frustum with the grid's bounding box to determine the first and last slice indices ($k_{min}, k_{max}$) along $\tilde{K}$ [S4, S5]. If the interval is empty, terminate.
5.  **Initialize Overlap Box:** Compute the initial overlap box $B(0)$ for the first slice by finding the minimum and maximum $\tilde{U}$ and $\tilde{V}$ coordinates of the frustum's entry and exit points with that slice [S4, S5].
6.  **Compute Increment Vector:** Calculate the constant vector $\Delta B = (du_{min}, du_{max}, dv_{min}, dv_{max})$, where the terms are the slopes of the bounding frustum planes in grid coordinate space [S4, S5].
7.  **Traverse Slices:** For each slice $k$ from $k_{min}$ to $k_{max}$:
    a.  **Update Overlap Box:** Incrementally compute the next slice's overlap box using $B(k+1) = B(k) + \Delta B$ [S4, S5]. This requires four floating-point additions [S4, S5].
    b.  **Compute Cell IDs:** Convert the floating-point coordinates of $B(k)$ to integer cell indices $[i_0..i_1] \times [j_0..j_1]$ by truncating the min/max $\tilde{U}$ and $\tilde{V}$ coordinates [S4, S5].
    c.  **Intersect:** Intersect all rays in the packet with all primitives (triangles or volume data) in the cells defined by $[i_0..i_1] \times [j_0..j_1]$ [S4, S5].
    d.  **Early Termination:** If all rays have intersected their targets, terminate traversal [S4, S5].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def coherent_grid_traversal(rays, grid_dims, grid_bounds):
    # 1. Transform rays to canonical grid space
    # rays: list of Ray objects with origin and direction
    # grid_dims: (Nx, Ny, Nz)
    # Transform function maps world coordinates to [0, Nx) x [0, Ny) x [0, Nz)
    canonical_rays = [transform_to_canonical(ray, grid_dims) for ray in rays]
    
    # 2. Determine major axis K
    # Select axis with max absolute direction component of the first ray
    first_ray_dir = canonical_rays[0].direction
    k_axis = argmax(abs(first_ray_dir))
    u_axis = (k_axis + 1) % 3
    v_axis = (k_axis + 2) % 3
    
    # 3. Compute Frustum Planes
    # For primary rays, use 4 corner rays to define the frustum
    # Calculate slopes of bounding planes in U and V directions along K
    # du_min, du_max, dv_min, dv_max are the increments per unit step in K
    du_min, du_max, dv_min, dv_max = compute_frustum_slopes(canonical_rays, k_axis, u_axis, v_axis)
    
    # 4. Clip to Grid
    # Determine k_min and k_max where frustum intersects grid bounds in K
    k_min, k_max = clip_frustum_to_grid_bounds(canonical_rays, k_axis, grid_bounds)
    if k_min > k_max:
        return [] # Frustum misses grid
    
    # 5. Initialize Overlap Box B(0)
    # Compute min/max U, V coordinates of frustum intersection with slice k_min
    b_u_min, b_v_min, b_u_max, b_v_max = compute_initial_overlap_box(canonical_rays, k_min, k_axis, u_axis, v_axis)
    
    # 6. Increment Vector
    delta_b = (du_min, du_max, dv_min, dv_max)
    
    visited_cells = []
    
    # 7. Traverse Slices
    for k in range(k_min, k_max + 1):
        # a. Update Overlap Box
        b_u_min += delta_b[0]
        b_u_max += delta_b[1]
        b_v_min += delta_b[2]
        b_v_max += delta_b[3]
        
        # b. Compute Cell IDs (Truncate)
        i0 = floor(b_u_min)
        i1 = floor(b_u_max)
        j0 = floor(b_v_min)
        j1 = floor(b_v_max)
        
        # Clamp to grid dimensions
        i0 = clamp(i0, 0, grid_dims[0] - 1)
        i1 = clamp(i1, 0, grid_dims[0] - 1)
        j0 = clamp(j0, 0, grid_dims[1] - 1)
        j1 = clamp(j1, 0, grid_dims[1] - 1)
        
        # c. Intersect
        # Yield cells and intervals for intersection
        for i in range(i0, i1 + 1):
            for j in range(j0, j1 + 1):
                # In volume rendering, check brick map for this (i, j, k)
                # In ray tracing, intersect rays with primitives in cell (i, j, k)
                yield (i, j, k)
                
        # d. Early Termination Check
        if all_rays_intersected(canonical_rays):
            break
```

**Edge Cases and Pitfalls:**
*   **Ray Divergence:** If rays in a packet span an angle greater than $\pi/2$, they will have different signs along the traversal axis, violating the front-to-back traversal assumption [S4, S5].
*   **Silhouette Hit Points:** If primary rays hit an object's silhouette, the resulting hit points can be distant, creating a wide frustum where the method breaks down [S5]. Mitigation: Split the packet into subpackets if hit points are too far apart [S4, S5].
*   **Roll Angle:** When the "roll" angle of the bundles to the volume is close to $\pi/4$, the number of block fetches can double [S1, S8, unofficial].
*   **Redundant Intersections:** Grids do not conform tightly to geometry, causing redundant triangle intersections. Large triangles or those straddling boundaries may appear in multiple cells [S4]. Use mailboxing to avoid re-intersection [S4, S5].
*   **Frustum Culling Overhead:** A triangle's bounding box may overlap traversed cells but not fall entirely outside the frustum. This overhead cannot be avoided by frustum culling [S4].

## Subtopics

### [Sphere-Center Method](5200-sphere-center-method.md)
An optimization for glyph-based particle visualization within CGT that eliminates data duplication by storing each particle in exactly one grid cell, using an enlarged frustum traversal.

### [Macrocell Hierarchical CGT](5464-macrocell-hierarchical-cgt.md)
Extends CGT to large volumes by introducing a recursive hierarchy of macrocells ($2^3$ children), amortizing traversal costs across multiple levels of granularity.

### [Slice Parallelisation](6292-slice-parallelisation.md)
A parallelization scheme for CGT where processing elements are assigned to slices of the volume, reducing redundant block fetches from neighboring bundles.

### [Bundle Parallelisation](6293-bundle-parallelisation.md)
A parallelization scheme for CGT where bundles of rays are processed in parallel, distributing work across processing elements.

### [SIMD Sphere/Frustum Culling](10012-simd-sphere-frustum-culling.md)
A culling technique for spheres in CGT that checks if the sphere center is contained within an enlarged bounding frustum, avoiding unnecessary intersection tests.

### [Hierarchical Grid with Macrocells](10019-hierarchical-grid-with-macrocells.md)
A hierarchical grid structure that divides densely populated regions more finely, using parameter range culling for aggressive early-out tests.

## Operating characteristics
**Time and Memory Complexity:**
*   **Traversal Cost:** The traversal cost per step becomes independent of the number of rays in the packet, encouraging larger packets [S4].
*   **Cell Visits:** Frustum traversal visits roughly 10 to 20 times fewer cells for $4 \times 4$ packets and over 50 times fewer for $8 \times 8$ packets compared to single-ray traversal [S5, S4].
*   **Memory:** The uniform grid is simple to build but less efficient than kd-trees for complex scenes. It requires storing the grid structure, which can be large for high-resolution volumes [S3, unofficial][S5].

**Measured Performance:**
*   On a dual 3.2 GHz Xeon, rendering a 174K triangle animated scene at 1024x1024 with shading/textures/shadows achieves 1.4 fps [S4, S5]. Excluding shading, it achieves 3.4 fps [S4, S5].
*   For a 16K triangle animated hand, it achieves 15.3 fps with shading/shadows and 34.5 fps without [S4, S5].
*   In the 'toys' scene (11K tris), mailboxing and frustum culling reduced ray-triangle intersections from 14.0M to 1.0M (factor of 14.0), compared to 0.82M for a kd-tree [S5].
*   CGT outperforms single-ray 3DDDA by a factor of 6 to 21, depending on the scene and viewpoint [S4, S5].
*   Sublinear scaling with image resolution is observed because higher resolutions enable larger packets [S2, S6].

**Strengths and Limitations:**
*   **Strengths:** Fast grid rebuilds make it ideal for dynamic/animated scenes [S3, unofficial][S4]. SIMD optimizations make the traversal step extremely cheap (2 instructions per slice) [S4, S6].
*   **Limitations:** Uniform grids are ill-suited for highly complex scenes with high variation in geometry size and density (e.g., Boeing 777, teapot-in-a-stadium) [S5, S4]. Performance degrades if the roll angle is close to $\pi/4$ [S1, S8, unofficial].

**When to Use:**
*   Use for dynamic scenes requiring per-frame grid rebuilds [S3, unofficial][S4].
*   Use for coherent ray packets where rays span an angle less than $\pi/2$ [S2, S4].
*   Avoid for highly complex, static scenes with high density variation where kd-trees or BVHs are more efficient [S5, S4].

## Background
Coherent Grid Traversal was introduced in 2006 by Wald, Ize, Kensler, Knoll, and Parker, published in ACM Transactions on Graphics (SIGGRAPH) [S4, S7]. It was developed to address the limitations of packet-based ray tracing on uniform grids, specifically the packet divergence problem that breaks cell-by-cell traversal algorithms like 3DDDA [S4, S5]. The method was initially designed for ray tracing animated scenes, leveraging the fast rebuildability of uniform grids [S4, S5]. It has since been adapted for volume rendering and particle visualization [S1, unofficial][S7].

## Sources
[S1] PPT - A Coherent Grid Traversal Algorithm for Volume Rendering PowerPoint Presentation - ID:991792, https://slideserve.com/kineta/a-coherent-grid-traversal-algorithm-for-volume-rendering-powerpoint-ppt-presentation, fetched 2026-10-24 (unofficial)
[S2] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S3] Ray Tracing Animated Scenes using Coherent Grid Traversal Ray Tracing Animated Scenes using Coherent Grid Traversal. I Wald, T Ize, A Kensler, A Knoll, S Parker SCI Institute, University of Utah. Cohe, https://www.slideserve.com/penn/ray-tracing-animated-scenes-using-coherent-grid-traversal, fetched 2026-10-25 (unofficial)
[S4] PDF Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~thiago/papers/grid.pdf, fetched 2026-10-25
[S5] PDF Ray Tracing Animated Scenes using Coherent Grid Traversal, https://www.sci.utah.edu/~wald/Publications/2006/CGT/grid.pdf, fetched 2026-10-25
[S6] A coherent grid traversal approach to visualizing particle-based simulation data, https://ieeexplore.ieee.org/abstract/document/4293019/, fetched 2026-10-25
[S7] A Coherent Grid Traversal Approach to Visualizing Particle ..., https://www.sci.utah.edu/~thiago/papers/pcgt_revised.pdf, fetched 2026-10-25
[S8] A Coherent Grid Traversal Algorithm for Volume Rendering, https://www.slideserve.com/iliana-house/a-coherent-grid-traversal-algorithm-for-volume-rendering, fetched 2026-10-26 (unofficial)
[S9] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
