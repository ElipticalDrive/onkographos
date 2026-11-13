# Voxel Raycasting with Normal Estimation

Also called: Amanatides and Woo algorithm, DDA Voxel Traversal, DDA-like voxel traversal, Digital Differential Analyzer voxel traversal, Parallel Ray Casting, Step-wise ray-sphere intersection for voxels, Voxel Raycasting via DDA, VoxelTrace

## Overview
Voxel Raycasting with Normal Estimation is a method for tracing a ray through a voxel grid to identify the first solid voxel intersected by the ray and estimating the surface normal at the entry point [S2, S3, unofficial]. It solves the problem of efficient collision detection and surface identification in discrete volumetric data by walking the grid directly, visiting only intersected cells with $O(\text{max\_distance})$ complexity, which is more efficient than general physics engine raycasts for voxel grids [S2, unofficial]. It sets itself apart from naive fixed-step marchers by using the Amanatides-Woo DDA traversal to step only when the ray crosses a voxel boundary, making it efficient for sparse or large grids, and from hierarchical methods by operating on a flat or chunked grid without precomputed acceleration structures [S1, S6, S3].

## How it works
The method combines the Amanatides-Woo voxel traversal algorithm to find the hit voxel with a heuristic normal estimation based on the ray's displacement vector relative to the voxel grid axes.

### Inputs and Outputs
**Inputs:**
*   `ray_origin`: The starting position of the ray in world coordinates ($\mathbb{R}^3$).
*   `ray_direction`: The normalized direction vector of the ray ($\mathbb{R}^3$).
*   `max_dist`: The maximum length of the ray cast ($\mathbb{R}$).
*   `grid`: The voxel data structure, providing access to voxel occupancy status (`solidAt(x, y, z)`) and potentially color/value data.
*   `epsilon`: A small constant to handle near-zero ray direction components to avoid division by zero or NaN issues. Typical value: $0.0001$ [S2, unofficial].

**Outputs:**
*   `hit`: Boolean indicating if a solid voxel was found.
*   `hit_pos`: The integer coordinates $(x, y, z)$ of the first solid voxel encountered.
*   `normal`: The estimated surface normal vector at the entry point of the hit voxel.

### Data Structures
The grid is typically stored as a 3D array or a collection of chunks. For the purpose of this method, we assume a function `solidAt(x, y, z)` returns `true` if the voxel at integer coordinates $(x, y, z)$ is occupied/solid, and `false` otherwise.

### Parameters
*   `epsilon`: $0.0001$ [S2, unofficial].
*   `max_dist`: Variable, e.g., $6.0$ [S3, unofficial].

### Algorithm Steps

1.  **Degeneracy Check**: If the squared magnitude of the ray direction is less than `epsilon` (or if the ray vector is effectively zero), return `false` immediately [S2, unofficial].
2.  **Initialization**:
    *   Calculate the step vector components: `step_x = sign(ray_direction.x)`, `step_y = sign(ray_direction.y)`, `step_z = sign(ray_direction.z)` [S2, unofficial].
    *   Determine the starting voxel coordinates: `voxel_x = floor(ray_origin.x)`, `voxel_y = floor(ray_origin.y)`, `voxel_z = floor(ray_origin.z)` [S2, S3, unofficial].
    *   Check if the starting voxel is solid. If `solidAt(voxel_x, voxel_y, voxel_z)` is true, calculate the entry normal based on the ray direction and local position within the voxel, then return `true` with this normal [S2, unofficial]. *Note: Some implementations skip this check and rely on the main loop, but checking the start voxel prevents missing hits if the ray originates inside a solid voxel.*
3.  **Delta Calculation**:
    *   Compute the reciprocal of the ray direction components for stepping. For each axis $i \in \{x, y, z\}$:
        *   If $|ray\_direction.i| > \epsilon$, `step_size_i = 1.0 / |ray_direction.i|`.
        *   Else, `step_size_i = infinity` [S2, unofficial].
    *   Calculate the distance to the first voxel boundary along each axis:
        *   If `step_x > 0`: `delta_x = (voxel_x + 1 - ray_origin.x) * step_size_x`.
        *   If `step_x < 0`: `delta_x = (ray_origin.x - voxel_x) * step_size_x`.
        *   Similarly for `delta_y` and `delta_z` [S2, unofficial].
    *   Initialize `delta` values to infinity for axes where the step size is infinity.
4.  **DDA Traversal Loop**:
    *   Initialize `prev_pos = ray_origin` [S3, unofficial].
    *   Loop while `t < max_dist`:
        *   Identify the axis with the smallest `delta` value. Tie-breaking priority is typically $X > Y > Z$ (i.e., if `delta_x == delta_y`, choose X) [S2, unofficial].
        *   Advance the voxel index along that axis:
            *   If X is smallest: `voxel_x += step_x`, `delta_x += step_size_x`.
            *   Else if Y is smallest: `voxel_y += step_y`, `delta_y += step_size_y`.
            *   Else: `voxel_z += step_z`, `delta_z += step_size_z`.
        *   Update `current_pos = ray_origin + ray_direction * delta_min` (where `delta_min` is the smallest delta before increment) [S3, G, unofficial]. *Note: The exact position calculation for normal estimation requires the position at the moment of entry into the new voxel.*
        *   Check if `solidAt(voxel_x, voxel_y, voxel_z)` is true.
        *   If solid:
            *   Set `hit_pos = (voxel_x, voxel_y, voxel_z)`.
            *   Calculate displacement vector `diff = current_pos - prev_pos` [S3, unofficial].
            *   Estimate the normal based on the largest component of `diff`:
                *   If $|diff.x| > |diff.y|$ and $|diff.x| > |diff.z|$:
                    *   `normal = (-sign(ray_direction.x), 0, 0)` [S3, unofficial].
                *   Else if $|diff.y| > |diff.z|$:
                    *   `normal = (0, -sign(ray_direction.y), 0)` [S3, unofficial].
                *   Else:
                    *   `normal = (0, 0, -sign(ray_direction.z))` [S3, unofficial].
            *   Return `true` with `hit_pos` and `normal`.
        *   Update `prev_pos = current_pos`.
        *   If `delta_min > max_dist`, break loop.
5.  **Result**: If the loop completes without finding a solid voxel, return `false` [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_raycast_with_normal(ray_origin, ray_direction, max_dist, grid, epsilon=0.0001):
    # 1. Degeneracy Check
    if ray_direction.length_squared() < epsilon:
        return False, None, None

    # 2. Initialization
    step_x = 1 if ray_direction.x > 0 else (-1 if ray_direction.x < 0 else 0)
    step_y = 1 if ray_direction.y > 0 else (-1 if ray_direction.y < 0 else 0)
    step_z = 1 if ray_direction.z > 0 else (-1 if ray_direction.z < 0 else 0)

    voxel_x = floor(ray_origin.x)
    voxel_y = floor(ray_origin.y)
    voxel_z = floor(ray_origin.z)

    # Check starting voxel
    if grid.solid_at(voxel_x, voxel_y, voxel_z):
        # Estimate normal for entry into starting voxel
        # This part is implementation-specific based on local position
        # For simplicity, we assume the main loop handles entry normals
        # or we return a default normal if ray starts inside.
        # See S2 for "calculate the entry normal by finding the closest face"
        pass 

    # 3. Delta Calculation
    if abs(ray_direction.x) > epsilon:
        step_size_x = 1.0 / abs(ray_direction.x)
    else:
        step_size_x = float('inf')

    if abs(ray_direction.y) > epsilon:
        step_size_y = 1.0 / abs(ray_direction.y)
    else:
        step_size_y = float('inf')

    if abs(ray_direction.z) > epsilon:
        step_size_z = 1.0 / abs(ray_direction.z)
    else:
        step_size_z = float('inf')

    if step_x > 0:
        delta_x = (voxel_x + 1 - ray_origin.x) * step_size_x
    else:
        delta_x = (ray_origin.x - voxel_x) * step_size_x

    if step_y > 0:
        delta_y = (voxel_y + 1 - ray_origin.y) * step_size_y
    else:
        delta_y = (ray_origin.y - voxel_y) * step_size_y

    if step_z > 0:
        delta_z = (voxel_z + 1 - ray_origin.z) * step_size_z
    else:
        delta_z = (ray_origin.z - voxel_z) * step_size_z

    prev_pos = ray_origin
    current_pos = ray_origin

    # 4. DDA Traversal Loop
    while True:
        # Find smallest delta
        if delta_x <= delta_y and delta_x <= delta_z:
            axis = 'x'
            delta_min = delta_x
        elif delta_y <= delta_z:
            axis = 'y'
            delta_min = delta_y
        else:
            axis = 'z'
            delta_min = delta_z

        if delta_min > max_dist:
            break

        # Advance voxel
        if axis == 'x':
            voxel_x += step_x
            delta_x += step_size_x
        elif axis == 'y':
            voxel_y += step_y
            delta_y += step_size_y
        else:
            voxel_z += step_z
            delta_z += step_size_z

        # Calculate position at entry of new voxel
        # Note: delta_min is the t-value of the boundary crossed
        current_pos = ray_origin + ray_direction * delta_min

        # Check solidity
        if grid.solid_at(voxel_x, voxel_y, voxel_z):
            # 5. Normal Estimation
            diff = current_pos - prev_pos
            
            abs_diff_x = abs(diff.x)
            abs_diff_y = abs(diff.y)
            abs_diff_z = abs(diff.z)

            if abs_diff_x > abs_diff_y and abs_diff_x > abs_diff_z:
                normal_x = -step_x
                normal_y = 0.0
                normal_z = 0.0
            elif abs_diff_y > abs_diff_z:
                normal_x = 0.0
                normal_y = -step_y
                normal_z = 0.0
            else:
                normal_x = 0.0
                normal_y = 0.0
                normal_z = -step_z

            return True, (voxel_x, voxel_y, voxel_z), (normal_x, normal_y, normal_z)

        prev_pos = current_pos

    return False, None, None
```

### Edge Cases and Pitfalls
*   **Large Step Size**: If the step size (distance between voxel boundaries) is large relative to the voxel size, the normal estimation based on `diff` may be inaccurate. The method assumes the ray enters the voxel face corresponding to the largest component of the displacement vector, which can fail if the ray passes through a corner or if the ray direction is nearly parallel to a grid axis [S3, unofficial].
*   **Degenerate Rays**: Rays with zero or near-zero direction components must be handled to avoid division by zero. The `epsilon` check handles this [S2, unofficial].
*   **Parallelism**: When implementing this in parallel (e.g., using OpenMP), incorrect management of inner loop variables (distinguishing between loop variables and invariants) can cause errors [S1, unofficial].
*   **Tie-Breaking**: The order of delta comparisons affects which axis is chosen when deltas are equal. A consistent tie-breaking rule (e.g., X > Y > Z) is required for deterministic results [S2, unofficial].

## Subtopics

### [DN_step_map](8596-dn-step-map.md)
A DDA algorithm that determines the sequence of voxels intersected by a ray without explicit geometric intersection tests, using precomputed boundary crossing distances. It sets itself apart by being reach-based and efficient for sparse grids, building on this method's traversal mechanism to identify the hit voxel and entry face.

### [Voxel Raycasting with DDA and Vertical Stripe Rendering](2206-voxel-raycasting-with-dda-and-vertical-stripe-rendering.md)
Variant focusing on rendering optimization via vertical stripes.

### [Wolfenstein 3D Raycasting](2366-wolfenstein-3d-raycasting.md)
Early 2D raycasting technique, not directly applicable to 3D voxel grids.

### [Voxel Shadow Raycasting with DDA](2410-voxel-shadow-raycasting-with-dda.md)
Variant for shadow ray computation.

### [Siddon's Algorithm](2590-siddon-s-algorithm.md)
Alternative ray traversal algorithm for line segments in grids.

### [3D Bresenham Voxel Tracing](3949-3d-bresenham-voxel-tracing.md)
Integer-based voxel traversal algorithm.

### [Restartable DDA for Voxel Traversal](5187-restartable-dda-for-voxel-traversal.md)
Variant allowing traversal restarts.

### [Naive Voxel Ray Traversal](5192-naive-voxel-ray-traversal.md)
Fixed-step marching approach.

### [DDA Skip](5260-dda-skip.md)
Optimization to skip empty regions.

### [Occupancy-aware DDA Traversal](5261-occupancy-aware-dda-traversal.md)
Variant using occupancy data to optimize traversal.

## Operating characteristics
*   **Time Complexity**: $O(\text{max\_distance})$ in the worst case, where the ray traverses every voxel along its path. In sparse grids, the number of visited voxels is proportional to the surface area of the object intersected, making it efficient [S2, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary space for the traversal algorithm itself, excluding the storage of the voxel grid [S1, unofficial].
*   **Performance**: On a 256x256x256 colorfield (approx. 16 million data points), a software-mode implementation achieves "decent speed" at 2000x2000 resolution. For dense objects at 800x600 resolution, it can achieve real-time rates [S1, unofficial]. It is more efficient than PhysX raycasts for voxel grids [S2, unofficial].
*   **Strengths**: Parallelizable (each ray is independent), efficient for sparse grids, simple implementation [S1, S2, unofficial].
*   **Limitations**: Performance degrades with dense objects or large empty spaces if no hierarchical structure is used. Normal estimation is heuristic and may be inaccurate for rays passing through corners or with large step sizes [S3, unofficial].
*   **When to Use**: Collision detection, shadow raycasting, and rendering in sparse voxel grids where hierarchical structures are not available or too costly to build [S1, S2, unofficial].
*   **When Not to Use**: Highly dense grids without acceleration structures, or when precise surface normals are required for complex geometries where heuristic estimation fails [S3, unofficial].

## Background
The method is based on the Amanatides-Woo algorithm, a classic ray traversal algorithm for voxel grids [S1, unofficial]. It was popularized in the context of voxel rendering for its efficiency and parallelizability [S1, unofficial]. The normal estimation component is a heuristic added to provide surface orientation information for shading or collision response, derived from the ray's displacement vector during traversal [S3, unofficial]. The approach has been implemented in various software renderers and voxel engines, often leveraging multi-threading for performance [S1, S2, unofficial].

## Sources
[S1] Voxel Renderer: Parallellism is key - Jeroen Baert's Blog, https://forceflow.be/2012/03/26/voxel-renderer-parallelism-is-key/, fetched 2026-10-24 (unofficial)
[S2] VektorKnight/vektor-voxels, https://github.com/VektorKnight/vektor-voxels, fetched 2026-10-27 (unofficial)
[S3] whudsontosh/CPU-Rasterizer, https://github.com/whudsontosh/CPU-Rasterizer, fetched 2026-10-27 (unofficial)
