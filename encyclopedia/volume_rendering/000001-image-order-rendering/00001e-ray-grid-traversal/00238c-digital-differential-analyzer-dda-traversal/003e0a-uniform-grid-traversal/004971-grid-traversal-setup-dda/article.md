# Grid Traversal Setup (DDA)
Also called: Ray marching grid setup, Voxel traversal initialization
## Overview
Grid Traversal Setup (DDA) initializes the parameters required for a 3D Digital Differential Analyzer (DDA) to traverse a voxel grid along a ray. It computes step directions, initial voxel indices, and distances to the next voxel boundaries, preparing the state for the iterative traversal loop. It differs from [Computational Space Traversal](10339-computational-space-traversal.md) by operating directly in the uniform grid's coordinate space, mapping ray origins and directions to voxel indices and boundary distances rather than abstract computational steps.
## How it works
### Inputs and Outputs
**Inputs:**
*   `ray`: A ray defined by origin `ray.origin` (float3), direction `ray.dir` (float3), and traversal interval `[tmin, tmax]`.
*   `aabb`: The axis-aligned bounding box of the voxel grid, defined by `aabb.min` and `aabb.max` (float3).
*   `res`: The resolution of the grid (integer3), representing the number of voxels along each axis.
*   `eps`: A small epsilon value to prevent numerical errors at boundaries.

**Outputs:**
*   `current_index`: The starting voxel index (int3).
*   `final_index`: The ending voxel index (int3).
*   `index_delta`: The step direction per axis (int3), values are 0 or 1.
*   `tdist`: The distance to the next voxel boundary along each axis (float3).
*   `step_float`: The signed step vector for traversal (float3), values are -1, 0, or 1.

### Data Structures
*   `float3`: A vector of three 32-bit floating-point numbers.
*   `int3`: A vector of three 32-bit integers.
*   `voxel_size`: A `float3` representing the physical size of a single voxel.
*   `inv_dir`: A `float3` representing the inverse of the ray direction components.

### Steps
1.  **Compute Voxel Size**: Calculate the physical dimensions of a single voxel by dividing the AABB dimensions by the resolution.
    $$ \text{voxel\_size} = \frac{\text{aabb.max} - \text{aabb.min}}{\text{res}} $$
2.  **Compute Ray Start and End Points**: Determine the physical start and end points of the ray segment within the traversal range, applying epsilon to avoid boundary issues.
    $$ \text{ray\_start} = \text{ray.origin} + \text{ray.dir} \times (\text{tmin} + \epsilon) $$
    $$ \text{ray\_end} = \text{ray.origin} + \text{ray.dir} \times (\text{tmax} - \epsilon) $$
3.  **Map Start Point to Voxel Index**: Convert the physical start point to a voxel index. This involves applying a contraction function (mapping physical coordinates to normalized grid coordinates) and scaling by resolution, then clamping to the grid bounds.
    $$ \text{current\_index} = \text{clamp}(\text{apply\_contraction}(\text{ray\_start}) \times \text{res}, 0, \text{res} - 1) $$
    *Note: `apply_contraction` maps physical coordinates to the range [0, 1] relative to the AABB.*
4.  **Map End Point to Voxel Index**: Similarly, convert the physical end point to a voxel index.
    $$ \text{final\_index} = \text{clamp}(\text{apply\_contraction}(\text{ray\_end}) \times \text{res}, 0, \text{res} - 1) $$
5.  **Determine Step Direction**: For each axis, determine the direction of traversal based on the ray direction component.
    $$ \text{index\_delta} = \text{make\_int3}(\text{ray.dir.x} > 0 ? 1 : 0, \text{ray.dir.y} > 0 ? 1 : 0, \text{ray.dir.z} > 0 ? 1 : 0) $$
6.  **Compute Distance to Next Boundary**: Calculate the distance from the ray start point to the next voxel boundary along each axis. This requires the inverse ray direction.
    $$ \text{inv\_dir} = \frac{1}{\text{ray.dir}} $$
    $$ \text{tmax\_xyz} = \left( (\text{aabb.min} + (\text{start\_index} \times \text{voxel\_size} - \text{ray\_start})) \times \text{inv\_dir} \right) + \text{tmin} $$
    *Note: `start_index` is `current_index`. The term `(start_index * voxel_size)` gives the physical start of the current voxel.*
7.  **Initialize Traversal Distances**: Handle rays parallel to grid axes to avoid division by zero. If a direction component is zero, cap the distance to `tmax`.
    $$ \text{tdist} = \text{make\_float3}(\text{ray.dir.x} == 0.0f ? \text{tmax} : \text{tmax\_xyz.x}, \text{ray.dir.y} == 0.0f ? \text{tmax} : \text{tmax\_xyz.y}, \text{ray.dir.z} == 0.0f ? \text{tmax} : \text{tmax\_xyz.z}) $$
8.  **Compute Step Float Vector**: Determine the signed step for each axis. If the direction is zero, the step is zero. Otherwise, it is +1 or -1.
    $$ \text{step\_float} = \text{make\_float3}(\text{ray.dir.x} == 0.0f ? 0.0f : (\text{ray.dir.x} > 0.0f ? 1.0f : -1.0f), \dots) $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_traversal_setup(ray_origin, ray_dir, tmin, tmax, aabb_min, aabb_max, res, eps):
    # 1. Compute Voxel Size
    voxel_size = (aabb_max - aabb_min) / res

    # 2. Compute Ray Start and End Points
    ray_start = ray_origin + ray_dir * (tmin + eps)
    ray_end = ray_origin + ray_dir * (tmax - eps)

    # 3. Map Start Point to Voxel Index
    # apply_contraction maps physical coord to [0, 1] range relative to AABB
    # contraction = (coord - aabb_min) / (aabb_max - aabb_min)
    start_contraction = (ray_start - aabb_min) / (aabb_max - aabb_min)
    current_index = clamp(start_contraction * res, 0, res - 1)

    # 4. Map End Point to Voxel Index
    end_contraction = (ray_end - aabb_min) / (aabb_max - aabb_min)
    final_index = clamp(end_contraction * res, 0, res - 1)

    # 5. Determine Step Direction
    index_delta = int3(
        1 if ray_dir.x > 0 else 0,
        1 if ray_dir.y > 0 else 0,
        1 if ray_dir.z > 0 else 0
    )

    # 6. Compute Distance to Next Boundary
    # Handle division by zero for inv_dir
    inv_dir = float3(
        1.0 / ray_dir.x if ray_dir.x != 0.0 else 0.0,
        1.0 / ray_dir.y if ray_dir.y != 0.0 else 0.0,
        1.0 / ray_dir.z if ray_dir.z != 0.0 else 0.0
    )

    # Physical start of the current voxel
    voxel_start_phys = aabb_min + current_index * voxel_size

    # Distance to next boundary in physical space
    dist_to_boundary_phys = voxel_start_phys - ray_start

    # Convert to ray parameter t
    # t = dist / dir, but handle zero dir
    tmax_xyz = float3(
        dist_to_boundary_phys.x * inv_dir.x + tmin if ray_dir.x != 0.0 else tmax,
        dist_to_boundary_phys.y * inv_dir.y + tmin if ray_dir.y != 0.0 else tmax,
        dist_to_boundary_phys.z * inv_dir.z + tmin if ray_dir.z != 0.0 else tmax
    )

    # 7. Initialize Traversal Distances
    tdist = float3(
        tmax if ray_dir.x == 0.0 else tmax_xyz.x,
        tmax if ray_dir.y == 0.0 else tmax_xyz.y,
        tmax if ray_dir.z == 0.0 else tmax_xyz.z
    )

    # 8. Compute Step Float Vector
    step_float = float3(
        0.0 if ray_dir.x == 0.0 else (1.0 if ray_dir.x > 0.0 else -1.0),
        0.0 if ray_dir.y == 0.0 else (1.0 if ray_dir.y > 0.0 else -1.0),
        0.0 if ray_dir.z == 0.0 else (1.0 if ray_dir.z > 0.0 else -1.0)
    )

    return current_index, final_index, index_delta, tdist, step_float

def clamp(value, min_val, max_val):
    return max(min_val, min(value, max_val))
```

### Parameters
*   `eps`: A small positive float (e.g., $10^{-6}$) to prevent numerical errors when the ray origin is exactly on a voxel boundary. Not given in the sources.

### Edge Cases and Pitfalls
*   **Zero Ray Direction Components**: If a component of `ray.dir` is zero, `inv_dir` would cause division by zero. The method handles this by checking `ray.dir.x == 0.0f` and capping `tdist` to `tmax` and setting `step_float` to 0 [S1, unofficial].
*   **Boundary Precision**: Using epsilon helps avoid sampling the same voxel twice or missing a voxel due to floating-point inaccuracies at boundaries [S1, unofficial].

## Subtopics
## Operating characteristics
*   **Time Complexity**: $O(1)$, as it involves a fixed number of arithmetic operations per ray.
*   **Memory Complexity**: $O(1)$, storing a few vectors per ray.
*   **Strengths**: Efficient initialization for uniform grids; robust handling of axis-aligned rays.
*   **Limitations**: Specific to uniform grids; requires pre-computation of voxel size and AABB.
*   **When to Use**: When traversing a uniform voxel grid using DDA.
*   **When Not to Use**: For non-uniform grids or hierarchical structures (use [Binary Grid Ray Traversal Initialization](12500-binary-grid-ray-traversal-initialization.md) or other methods).

## Background
This method is part of the standard DDA ray traversal technique for uniform grids, commonly used in volume rendering and ray tracing. The specific implementation details are derived from unofficial sources [S1, unofficial].

## Sources
[S1] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
