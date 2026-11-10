# Voxel Grid Raycasting

Also called: DDA Ray Tracing, Ray Marching through Voxel Grid, Uniform Grid Raycasting, Voxel Ray Tracing with Voxel Marching, Voxel-Volume-Raycasting, Voxelization Raycasting, raycastVoxels

## Overview
Voxel Grid Raycasting is a rendering technique that stores scene geometry in a 3D voxel grid and traces rays through the grid by stepping from voxel to voxel to determine intersections and surface normals [S1, S2, S5, unofficial]. It solves the problem of efficiently determining ray-voxel intersections without explicit geometric intersection tests by using an analog of the Digital Differential Analyzer (DDA) algorithm [S2, S3, unofficial]. Unlike standard volume raycasting which samples continuous scalar fields, voxel raycasting operates on discrete integer coordinates, rounding float positions to the nearest voxel index to check occupancy [S3, unofficial]. It differs from hierarchical methods like [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) by using a fixed uniform grid, offering high cache efficiency on GPUs for uniform data but requiring $O(N^3)$ memory for dense volumes [G].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin $O$, ray direction $D$ (normalized), voxel grid dimensions ($W, H, D$), voxel size $s$, and the voxel data buffer.
*   **Outputs:** Hit position, voxel coordinates, surface normal, and material/color data. If no hit is found within the max distance or bounds, the ray returns a miss (often contributing to sky/ambient color).

**Data Structures:**
The voxel grid is typically stored in a 1D storage buffer or array.
*   **Layout:** A 1D array of integers (e.g., `u32` or `int`).
*   **Indexing:** For a grid with width $W$ and height $H$, the index for voxel $(x, y, z)$ is calculated as:
    $$ index = x + y \cdot W + z \cdot (W \cdot H) $$
    [S1, unofficial]
    Alternatively, some implementations use row-major ordering with specific strides, e.g., `(x * W + z) * H + y` [S2, unofficial].
*   **Content:** A value of `0` indicates an empty voxel. Non-zero values indicate occupied voxels, potentially encoding material properties or color (e.g., packed 8-bit channels) [S1, S2, unofficial].

**Parameters:**
*   `voxel_size` ($s$): Physical size of a voxel in world units. Typical value: $0.1$ [S1, unofficial].
*   `epsilon`: Small offset to prevent self-intersection. Typical value: $s \times 0.01$ [S1, unofficial].
*   `max_dist`: Maximum ray length. Typical value: $16.0$ [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Normalize the ray direction: $d = \frac{D}{\|D\|}$ [S2, unofficial].
    *   Calculate the starting voxel coordinates by flooring the start position divided by the bin size:
        $$ current\_voxel = \lfloor \frac{O}{s} \rfloor $$
        [S2, unofficial]
    *   Determine traversal steps ($\pm 1$) for each axis based on the sign of the ray direction components:
        $$ step_x = \begin{cases} 1 & \text{if } d_x \ge 0 \\ -1 & \text{otherwise} \end{cases} $$
        (Similarly for $y$ and $z$) [S2, unofficial].
    *   Calculate the distance to the next voxel boundary for each axis. The target boundary coordinate is $(current\_voxel + step) \times s$.
        $$ t_{max} = \begin{cases} \frac{boundary - O}{d} & \text{if } d \neq 0 \\ \infty & \text{if } d = 0 \end{cases} $$
        [S2, S6, unofficial]
        *Note:* If a ray component is exactly 0, $t_{max}$ and $t_{delta}$ are set to a minimal value (e.g., `FLT_MIN`) to effectively remove that axis from the traversal comparison logic [S2, unofficial].
    *   Calculate the step delta (distance along the ray to traverse one full voxel width):
        $$ t_{delta} = \frac{s}{|d|} \times step $$
        [S2, unofficial]
    *   Handle negative ray direction edge cases: If the ray starts in a voxel different from the last voxel and moves in a negative direction, decrement the corresponding voxel coordinate to ensure correct traversal start [S2, unofficial].

2.  **Traversal Loop:**
    *   Iterate while the current voxel is within bounds and has not exceeded the max distance or the target end voxel.
    *   Find the axis with the minimum $t_{max}$ value. This is the axis where the ray crosses the next grid plane.
    *   Increment the grid position on that axis by the corresponding step ($step_x, step_y, or step_z$).
    *   Update the distance traveled to the current $t_{max}$ value.
    *   Increment the $t_{max}$ for that axis by the corresponding $t_{delta}$ to move to the next boundary on that axis.
    *   Check for occupancy at the new `current_voxel` position.
    *   If occupied:
        *   Calculate the surface normal based on the axis of entry (e.g., if entered via X, normal is $\pm \hat{x}$).
        *   Return hit data (position, normal, voxel index).
    *   If empty: Continue to the next iteration.

3.  **Ray Bouncing (Optional):**
    *   If the hit voxel is opaque and further bounces are required:
        *   Spawn a new ray from the hit position: $O_{new} = O_{hit} + \epsilon \cdot N$.
        *   Calculate a new direction (reflection or random diffuse).
        *   Re-initialize traversal state for the new ray.
        *   Repeat until max bounces or transparency.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def raycast_voxel_grid(ray_origin, ray_direction, voxel_grid, voxel_size, max_dist):
    """
    Traces a ray through a uniform voxel grid using DDA traversal.
    
    Args:
        ray_origin: (x, y, z) float tuple
        ray_direction: (x, y, z) float tuple (assumed normalized)
        voxel_grid: 1D list/array of integers representing the grid
        voxel_size: float, size of one voxel side
        max_dist: float, maximum travel distance
        
    Returns:
        dict with 'hit', 'position', 'voxel', 'normal', or None
    """
    # 1. Initialization
    inv_voxel_size = 1.0 / voxel_size
    current_voxel = (
        int(math.floor(ray_origin[0] * inv_voxel_size)),
        int(math.floor(ray_origin[1] * inv_voxel_size)),
        int(math.floor(ray_origin[2] * inv_voxel_size))
    )
    
    # Determine step direction
    step = (
        1 if ray_direction[0] >= 0 else -1,
        1 if ray_direction[1] >= 0 else -1,
        1 if ray_direction[2] >= 0 else -1
    )
    
    # Calculate tMax: distance to next boundary
    # Boundary coordinate = (current_voxel + step) * voxel_size
    boundaries = (
        (current_voxel[0] + step[0]) * voxel_size,
        (current_voxel[1] + step[1]) * voxel_size,
        (current_voxel[2] + step[2]) * voxel_size
    )
    
    # Handle zero direction components to avoid division by zero
    t_max = []
    t_delta = []
    for i in range(3):
        if ray_direction[i] != 0:
            t_max.append((boundaries[i] - ray_origin[i]) / ray_direction[i])
            t_delta.append(voxel_size / abs(ray_direction[i]) * step[i])
        else:
            t_max.append(float('inf'))
            t_delta.append(float('inf'))
            
    # Handle negative ray start adjustment if necessary (S2)
    # This ensures we start at the correct voxel if the ray enters from negative direction
    # Simplified: The floor operation usually handles this, but explicit checks exist in some implementations.
    
    dist = 0.0
    
    # 2. Traversal Loop
    # Determine bounds to prevent infinite loops
    grid_dims = get_grid_dimensions(voxel_grid) # Assumed helper
    
    while dist < max_dist:
        # Check bounds
        if not is_within_bounds(current_voxel, grid_dims):
            return None # Out of bounds
            
        # Check occupancy
        idx = voxel_index(current_voxel, grid_dims)
        if voxel_grid[idx] != 0:
            # Hit found
            # Calculate precise hit position using t_max of the axis we just crossed
            # The axis with the minimum t_max before the step was the crossing axis
            # However, in the loop structure below, we update t_max AFTER stepping.
            # We need to track the axis of the last step.
            
            # For simplicity, assume we calculate normal based on the axis that caused the step
            # In the loop below, 'axis' is determined by argmin(t_max)
            
            normal = get_normal_from_axis(axis, step[axis])
            hit_pos = ray_origin + ray_direction * dist
            return {
                'hit': True,
                'position': hit_pos,
                'voxel': current_voxel,
                'normal': normal,
                'value': voxel_grid[idx]
            }
        
        # Find axis with minimum t_max
        axis = 0
        min_t = t_max[0]
        for i in range(1, 3):
            if t_max[i] < min_t:
                min_t = t_max[i]
                axis = i
        
        # Update distance and position
        dist = min_t
        current_voxel = list(current_voxel)
        current_voxel[axis] += step[axis]
        current_voxel = tuple(current_voxel)
        
        # Update t_max for the crossed axis
        t_max[axis] += t_delta[axis]
        
        # Safety break for infinite loops in degenerate cases
        if dist > max_dist:
            break
            
    return None

def voxel_index(v, dims):
    # Assuming dims = (W, H, D) and layout x + y*W + z*W*H
    W, H, D = dims
    return v[0] + v[1] * W + v[2] * (W * H)

def is_within_bounds(v, dims):
    return 0 <= v[0] < dims[0] and 0 <= v[1] < dims[1] and 0 <= v[2] < dims[2]

def get_normal_from_axis(axis, step):
    normal = [0.0, 0.0, 0.0]
    normal[axis] = -step # Normal points against the step direction (into the voxel)
    return tuple(normal)
```

## Subtopics

### [Voxel-Cell Duality Mapping](2240-voxel-cell-duality-mapping.md)
Establishes a dual relationship between voxels and cells by logically shifting scalar positions, allowing ray tracers to intersect 8-voxel cell primitives without expanding storage.

### [Spherical Volume Ray Walking](3865-spherical-volume-ray-walking.md)
Extends Cartesian DDA to spherical coordinate grids, handling curved boundaries using line-sphere intersections for radial steps.

### [Plane Marching](3959-plane-marching.md)
Advances a ray by stepping to the nearest intersection with integer grid planes, simple but fragile in empty space as it never skips more than one voxel per step.

### [RLE4 Ray Tracing](5228-rle4-ray-tracing.md)
Traverses volume data compressed with Run-Length Encoding in the Y-dimension, using texture-based lookups to skip uncompressed surface regions.

### [Sparse Shadow Volume Raycast](5632-sparse-shadow-volume-raycast.md)
A hierarchical algorithm that dynamically adjusts traversal step size and LOD based on occupancy bits to skip empty space efficiently.

### [Regular Grid Traversal](5939-regular-grid-traversal.md)
Steps through a uniform 3D grid of cells, skipping empty space without primitive tests, offering high cache efficiency but $O(N^3)$ memory.

### [Voxel Raytracer](6403-voxel-raytracer.md)
Implements a full path-tracing loop with diffuse scattering and anti-aliasing, differing from simple direct illumination raycasting.

### [DirectVoxGO Volume Rendering](6412-directvoxgo-volume-rendering.md)
Accelerates neural radiance field training by sampling points directly on a structured voxel grid using a mask cache to skip free space.

### Voxel Resolve Search
A search algorithm used to find the pixel's voxel again in a brick named by the visibility buffer, stopping at the first cell whose hit matches the stored depth.

### Narrow Band Voxelization
A strategy that immediately returns a hit when reaching a narrow band voxel, resulting in a voxelization of the scene without using a solver.

### Dense Octree Accelerated Raymarching
Uses a dense octree stored in a 3D texture to skip empty space during ray traversal, switching to fine-grained voxel textures at leaf nodes.

### Voxel Resolve Search
A search algorithm used by voxel resolve and GI update to find the pixel's voxel again in the brick named by the visibility buffer [S7, unofficial]. It finds the cell holding the ray's point at the stored depth, then the two cells along each axis whose centers lie nearest that point, stopping at the first cell whose hit gives the stored depth again, bit for bit [S7, unofficial]. A voxel's test depends only on the ray and the voxel, ensuring every pass meets it at the same $t$ [S7, unofficial].

### Narrow Band Voxelization
A narrow band strategy that does not use any solver [S8]. When the algorithm reaches a narrow band voxel, it immediately returns a hit, resulting in a voxelization of the scene [S8].

### Dense Octree Accelerated Raymarching
An acceleration structure using a dense octree stored in a 3D texture to skip empty space during ray traversal [S9, unofficial]. The raymarcher first traverses the octree texture; once it hits a leaf node (occupied octant), it switches to the fine-grained voxel texture for detailed collision detection [S9, unofficial]. The octree is stored as a 3D texture (`texture_3d<u32>`) with data for different levels interleaved in the Z dimension [S9, unofficial]. This approach is significantly faster for ray traversal than linear buffers with Morton curve ordering because graphics hardware is optimized for spatially local texture access [S9, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels traversed along the ray. In the worst case (ray passes through the entire grid), $N$ is proportional to the grid dimension.
*   **Memory Complexity:** $O(W \cdot H \cdot D)$ for the voxel grid storage. This is a significant limitation for large, sparse volumes compared to hierarchical methods.
*   **Performance:** Highly efficient on GPUs due to regular memory access patterns and simple arithmetic operations [S1, S9, unofficial]. Cache coherency is high for uniform grids.
*   **Strengths:** Simple implementation, no complex data structure traversal overhead, deterministic, fast for dense volumes.
*   **Limitations:** High memory usage for large volumes, inefficient for sparse scenes (many empty voxels traversed), precision issues with very large grids or small voxels (floating point errors in $t_{max}$ accumulation).
*   **When to use:** Dense voxel data, real-time rendering on GPUs, simple collision detection.
*   **When not to use:** Sparse large-scale scenes, memory-constrained environments, scenes requiring high precision over vast distances.

## Background
Voxel grid raycasting is based on the Digital Differential Analyzer (DDA) algorithm for grid traversal, originally described by Amanatides and Woo in 1987 [S3, unofficial]. It is a fundamental technique in voxel-based rendering, often used in games and real-time applications where performance is critical. Variants like [Sparse Voxel Traversal](1525-sparse-voxel-traversal.md) and [Octree Traversal](1587-octree-traversal.md) were developed to address the memory inefficiencies of uniform grids.

## Sources
[S1] newDINO/voxel_ray_demo, https://github.com/newDINO/voxel_ray_demo, fetched 2026-10-24 (unofficial)
[S2] SamCZ/RayTracer, https://github.com/SamCZ/RayTracer, fetched 2026-10-24 (unofficial)
[S3] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S4] michschw/GpuPro3Vox, https://github.com/michschw/GpuPro3Vox, fetched 2026-10-26 (unofficial)
[S5] voxel/voxel-engine-stackgl, https://github.com/voxel/voxel-engine-stackgl, fetched 2026-10-27 (unofficial)
[S6] Kurble/amethyst_voxel, https://github.com/Kurble/amethyst_voxel, fetched 2026-10-27 (unofficial)
[S7] feat(render): voxels splatted to the pixels their footprints cover (ADR-0061), https://github.com/SugalAbubaker/nepheline/pull/115, fetched 2026-10-27 (unofficial)
[S8] Rendering piecewise approximations of SDFs through analytic intersections, https://doi.org/10.1016/j.cag.2024.103981, fetched 2026-10-27
[S9] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24 (unofficial)
