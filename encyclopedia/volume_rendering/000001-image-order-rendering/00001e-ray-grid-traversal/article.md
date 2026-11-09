# Ray-grid traversal

Also called: Voxel traversal, Grid traversal, DDA traversal, Amanatides-Woo traversal

## Overview
Ray-grid traversal determines the sequence of uniform grid cells (voxels or tiles) intersected by a ray, enabling efficient sampling and compositing in volume rendering [S2, S3, unofficial]. It solves the problem of enumerating grid intersections without missing thin cells or performing redundant work [S2, unofficial]. Unlike hierarchical methods that skip empty space via tree structures, standard ray-grid traversal steps through every cell boundary along the ray path [S1, unofficial][S2, unofficial]. It differs from [Acceleration](18-acceleration.md) by focusing on the geometric stepping logic itself, and from [Direct volume ray casting](19-direct-volume-ray-casting.md) by providing the traversal indices required for the sampling phase [G].

## Shared principles
All methods in this family share the core mechanism of advancing analytically from one cell boundary to the next, avoiding fixed-distance sampling steps [S5][S7, unofficial]. The algorithm determines the next cell by comparing the distances to the upcoming boundaries on the x, y, and z axes, selecting the minimum distance to advance [S2, unofficial][S3, unofficial][S5].

### Mathematical Foundation
Let the ray be defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized or unnormalized) [G]. The grid is defined by cell boundaries. For a uniform grid with voxel size $v$ and dimensions $N_x, N_y, N_z$, the traversal relies on three key precomputed values per axis $i \in \{x, y, z\}$ [S2, unofficial][S3, unofficial][S5]:

1.  **Step Direction ($\text{step}_i$):** Determines the index increment direction [S4, unofficial].
    $$
    \text{step}_i = \begin{cases} 
    +1 & \text{if } d_i > 0 \\
    -1 & \text{if } d_i < 0 \\
    0 & \text{if } d_i = 0 
    \end{cases}
    $$
    If $d_i = 0$, the ray never crosses a boundary in that dimension, and the corresponding distance is treated as infinity [S5].

2.  **Time/Distance Delta ($\Delta t_i$ or $s_{\Delta i}$):** The distance along the ray to traverse one voxel width [S2, unofficial][S5].
    $$
    \Delta t_i = \frac{v_i}{|d_i|} \cdot \text{step}_i \quad \text{or} \quad s_{\Delta i} = \frac{\Delta x_i}{|\widehat{d}_i|}
    $$
    where $v_i$ is the voxel size and $\widehat{d}_i$ is the unit direction component [S2, unofficial][S5].

3.  **Time/Distance to Next Boundary ($t_{max, i}$ or $s_{max, i}$):** The distance from the ray origin to the first boundary crossed in axis $i$ [S2, unofficial][S3, unofficial][S5].
    $$
    t_{max, i} = \frac{\text{boundary}_i - o_i}{d_i}
    $$
    The initial boundary is determined by the starting voxel index [S3, unofficial]. If the ray starts inside the grid, the initial boundary is the next one in the direction of travel [S2, unofficial][S3, unofficial][S5].

### Traversal Loop
The core loop iterates while the ray remains within the grid bounds [S5]:
1.  Identify the axis $k$ with the minimum boundary distance: $t_{min} = \min(t_{max, x}, t_{max, y}, t_{max, z})$ [S2, unofficial][S3, unofficial][S5].
2.  Record the current cell and the interval $[t_{entry}, t_{min}]$ [S1, unofficial][S5].
3.  Advance the index in axis $k$: $\text{index}_k \leftarrow \text{index}_k + \text{step}_k$ [S2, unofficial][S3, unofficial][S5].
4.  Update the boundary distance for axis $k$: $t_{max, k} \leftarrow t_{max, k} + \Delta t_k$ [S2, unofficial][S3, unofficial][S5].
5.  Update $t_{entry} \leftarrow t_{min}$ [S5].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dda_traversal(ray_origin, ray_direction, voxel_size, grid_bounds):
    # Initialize current voxel index
    current_voxel = floor(ray_origin / voxel_size)
    current_voxel = clamp(current_voxel, grid_bounds.min, grid_bounds.max)
    
    # Step direction per axis
    step = sign(ray_direction)
    
    # Distance to next boundary
    next_boundary = (current_voxel + step) * voxel_size
    t_max = (next_boundary - ray_origin) / ray_direction
    
    # Distance delta per axis
    t_delta = voxel_size / abs(ray_direction)
    
    # Handle zero direction components
    for i in range(3):
        if ray_direction[i] == 0:
            t_max[i] = float('inf')
            t_delta[i] = 0
            
    t_entry = 0.0
    
    while current_voxel is within grid_bounds:
        # Find minimum boundary distance
        min_idx = argmin(t_max)
        t_exit = t_max[min_idx]
        
        # Record cell and interval
        yield current_voxel, t_entry, t_exit
        
        # Advance index and update boundary
        current_voxel[min_idx] += step[min_idx]
        t_max[min_idx] += t_delta[min_idx]
        t_entry = t_exit
```

## Subtopics

### [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md)
Accelerates rendering by traversing a multi-dimensional hierarchy to adaptively select resolution levels based on error tolerance.

### [Woodcock Tracking with Macrocell Majorant](986-woodcock-tracking-with-macrocell-majorant.md)
A stochastic technique for unstructured tetrahedral meshes using coarse mesh hierarchies to bound extinction.

### [Front-to-Back Layer Traversal](1212-front-to-back-layer-traversal.md)
Processes Matryoshka layers in front-to-back order to handle occlusion correctly.

### [Dual Grid Traversal](1233-dual-grid-traversal.md)
Generates a manifold mesh by traversing a dual grid constructed from a primal voxel grid.

### [Backward Projection](1491-backward-projection.md)
Maps 3D volumetric data to a 2D image plane by traversing rays from pixels into the volume.

### [Lazy Sweep Ray Casting](1649-lazy-sweep-ray-casting.md)
Accelerates rendering of irregular grids by exploiting spatial coherence via a sweep-plane paradigm.

### [VolumeHDDA](2353-volumehdda.md)
Combines analytical DDA logic with tree-aware node skipping for sparse volume trees.

### [Ray Marching with Octant Masking](2488-ray-marching-with-octant-masking.md)
Advances ray origin in discrete steps using a precomputed octant mask to track traversal state.

### [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md)
Iterates through levels of a grid mipmap to efficiently visit child voxels.

### [Coherent Grid Traversal](2987-coherent-grid-traversal.md)
Processes bundles of coherent rays through uniform grids slice-by-slice.

### [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md)
Determines the sequence of uniform grid cells intersected by a ray.

### [Z-order curve traversal with run-length encoding](3408-z-order-curve-traversal-with-run-length-encoding.md)
Traverses a uniform voxel grid in Z-order while compressing identical runs.

### [Hero-like 4-Connectivity (H4T)](3436-hero-like-4-connectivity-h4t.md)
Guarantees strict 4-connectivity by resolving boundary crossing ambiguities via stable sorting.

### [Octree and Tree Traversal](4134-octree-and-tree-traversal.md)
Determines the sequence of nodes in a sparse voxel octree intersected by a ray.

### [Multi-Grid Ray Traversal](4142-multi-grid-ray-traversal.md)
Traverses rays through multiple binary occupancy grids to generate ray intervals.

### [Ray Template Traversal](7802-ray-template-traversal.md)
Stores traversal steps as a ray template to accelerate parallel rays [Yagel-1992-DRT].

### [Data Structures and Utilities](8868-data-structures-and-utilities.md)

### [Ray Marching Optimization](10233-ray-marching-optimization.md)
Accelerates rendering by replacing fixed-distance steps with analytical traversal.

### [Hardware-Specific Optimization](10277-hardware-specific-optimization.md)
Adapts traversal algorithms to exploit GPU/SIMD architectural features [G].

### Barycentric Coordinate Traversal
An alternative traversal algorithm that uses barycentric coordinates (u, v, w, x) to determine the child tetrahedron, avoiding explicit plane tests [S10, unofficial]. The loop continues while `childCounts[tetID] > 0` and `rLevel ≤ maxDepth`. Barycentric coordinates are calculated as $u = \text{volume}(point,l,r,b)/\text{vol}$, $v = \text{volume}(point,t,b,r)/\text{vol}$, $w = \text{volume}(point,t,l,b)/\text{vol}$, and $x = 1 - (u + v + w)$ [S10, unofficial]. The child index is computed using boolean logic: `tet = (RB × 4) + ((RB == TB) × 2) + inner`, where `RB` is `(u + v) < 0.5`, `TB` is `(v + w) < 0.5`, and `inner` is `(u < 0.5) & (v < 0.5) & (w < 0.5) & (x < 0.5)` [S10, unofficial].

### Perspective-Correct AMR Patch Traversal
A hybrid CPU-GPU traversal strategy where the CPU traverses the KD-tree of AMR patches back-to-front, and the GPU traverses cells within a patch [S11]. Sub-patches are generated to ensure perspective-correct sorting of cells relative to the observer. The vertex shader modifies each index according to the current observer position to generate proper cell ordering [S11].

### Majorant Supergrid Traversal
A medium traversal structure using a supergrid to allow empty and thin regions to be crossed in far fewer null collisions than paying for the densest voxel in the volume [S12, unofficial]. The `Medium::sample_real_interaction` consumes null collisions in one fused C++ walk instead of iterating them in Python, sharing a single loop with the DDA traversal when a supergrid is used [S12, unofficial]. It is faster than the global majorant approach because it avoids paying `optixTrace` for every null collision [S12, unofficial]. The `majorant_resolution_factor` controls supergrid resolution (default 8), and `majorant_factor` is a safety factor (default 1.2) [S12, unofficial]. The 'homogeneous' medium type explicitly rejects the supergrid parameter [S12, unofficial].

### Grid Traversal Setup (CUDA)
A CUDA device function that initializes the parameters for stepping through a voxel grid along a ray [S7, unofficial].

### Target Value Classification and Parameterization
A method to determine if a target isosurface lies within a ray segment defined by voxel entry/exit points, and to calculate its relative position for opacity/color contribution [S13, unofficial]. Trilinear interpolation is used to get values at entry ($V_{in}$) and exit ($V_{out}$). Target values are classified into three categories: (1) strictly between $V_{in}$ and $V_{out}$, (2) one endpoint within allowable window, (3) neither endpoint within window. For Category 1, the relative distance $t$ is parameterized using linear interpolation. For Category 2, $t$ is set to 0 or 1. Values are sorted by depth and effects calculated [S13, unofficial].

### Ray Connectivity
A property defining the set of cells a ray can traverse to from its current cell, inspired by pixel connectivity in image processing [S14, unofficial]. In 2D, there are 4-connectivity (orthogonal neighbors) and 8-connectivity (including diagonal neighbors) [S14, unofficial]. The Taxicab metric is $|x_1 - x_2| + |y_1 - y_2|$ and the Chebyshev metric is $\max(|x_1 - x_2|, |y_1 - y_2|)$ [S14, unofficial].

### Block Walking
An algorithm that traverses all blocks a ray intersects with, starting from closest to the camera, to measure the direct impact of block-walking on performance by writing to a scalar at each block visit [S15, unofficial]. Block walking reduces the number of queries (acceleration structure accesses) by 5-10x compared to standard ray tracing, translating to a 1.5x average performance improvement in block traversal mode [S15, unofficial]. However, when content becomes too sparse, block walking is not effective and can have a high cost [S15, unofficial]. In blend mode, performance improvements shrink due to costly access to individual voxel values [S15, unofficial].

### Adaptive grid traversal dispatcher
A dispatcher function that selects the appropriate traversal variant based on the maximum range of the segment [S16, unofficial]. It computes absolute deltas $dx = |end.x - start.x|$ and $dy = |end.y - start.y|$, then `maxRange = max(dx, dy)`. If `maxRange < 256.0f`, it calls `traverseGridSegment16` (Q8.8 fixed-point); otherwise, it calls `traverseGridSegment32` (Q24.8 fixed-point) [S16, unofficial].

### Regular Grid (rgridb) Renderer
A renderer for regular grids (rgridb) that separates primitive generation and rendering into two distinct steps, regenerating all primitives if the volume changes [S17, unofficial]. Unlike the hgridf renderer, it does not support partial updates of dirty cells; it regenerates the entire primitive set upon volume change [S17, unofficial].

### Ray-Plane-based Traversal
All processors accumulate interpolated values along a ray independently. Contributions are sorted with respect to visibility order before composing the final image [S18, unofficial]. Frame rate scales linearly with the number of processors [S18, unofficial].

## Background
The foundational algorithm, known as the Amanatides-Woo algorithm or 3D DDA, was introduced by J. Amanatides and A. Woo in 'A Fast Voxel Traversal Algorithm for Ray Tracing', published in Eurographics '87 [S2, unofficial][S5]. It was designed to efficiently list all voxel IDs traversed by a ray in a 3D grid [S2, unofficial]. The method has been widely implemented in various forms, including in real-time procedural voxel renderers like Minecraft-style raycasting [S6, unofficial], and in scientific computing tools like FIDASIM for collision detection [S5]. Modern implementations often optimize this core logic for hardware-specific constraints such as SIMD parallelism and cache hierarchy [S10277-hardware-specific-optimization.md].

## Sources
[S1] jw3126/VoxelRayTracers.jl, https://github.com/jw3126/VoxelRayTracers.jl, fetched 2026-10-24 (unofficial)
[S2] francisengelmann/fast_voxel_traversal, https://github.com/francisengelmann/fast_voxel_traversal, fetched 2026-10-24 (unofficial)
[S3] Victorique-GOSICK/Voxel-3D-DDA, https://github.com/Victorique-GOSICK/Voxel-3D-DDA, fetched 2026-10-24 (unofficial)
[S4] Super Fast Ray Casting in Tiled Worlds using DDA, https://www.youtube.com/watch?v=NbSee-XM7WA, fetched 2026-10-25 (unofficial)
[S5] 2026_surface_mesh_tools/docs/collision_detection/04_ray_grid_traversal ..., https://github.com/canesesmarjf/2026_surface_mesh_tools/blob/main/docs/collision_detection/04_ray_grid_traversal.md, fetched 2026-10-25
[S6] Minecraft Voxel DDA Raycast - Shadertoy, https://www.shadertoy.com/view/WccyWl, fetched 2026-10-26 (unofficial)
[S7] nerfstudio-project/nerfacc, https://github.com/nerfstudio-project/nerfacc, fetched 2026-10-26 (unofficial)
[S8] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S9] Linirby/HelloVoxel, https://github.com/Linirby/HelloVoxel, fetched 2026-10-27 (unofficial)
[S10] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S11] High-Quality, Semi-Analytical Volume Rendering for AMR ..., https://people.freedesktop.org/~marcheu/vis09-amr.pdf, fetched 2026-10-26
[S12] Sample matching for extinction gradients (review branch), https://github.com/AuroraRyan0301/mitsuba3/pull/1, fetched 2026-10-26 (unofficial)
[S13] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
[S14] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S15] [PDF] Sparse Volume Rendering using Hardware Ray Tracing and Block ..., https://gpuopen.com/download/SA2021_BlockWalk.pdf, fetched 2026-10-26 (unofficial)
[S16] FastLED: fl/traverse_grid.h Source File, https://fastled.io/docs/d9/d42/traverse__grid_8h_source.html, fetched 2026-10-30 (unofficial)
[S17] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
[S18] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
