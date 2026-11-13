# Grid Traversal Setup (DDA)

Also called: Ray marching grid setup, Voxel traversal initialization

## Overview
Grid Traversal Setup (DDA) initializes the parameters required for a 3D Digital Differential Analyzer (DDA) to traverse a voxel grid along a ray [S2, unofficial]. It computes the step directions, initial voxel indices, and distances to the next voxel boundaries for the ray [S2, unofficial]. This method sets the stage for the iterative traversal loop, ensuring that the ray correctly enters the grid and aligns with the voxel lattice [S1, S2, unofficial]. It differs from computational space traversal by operating directly in the grid's coordinate system and from one-pass over-allocated methods by computing exact boundary crossings rather than pre-allocating a fixed set of samples [S1, S2, unofficial].

## How it works
**Inputs:**
- `ray.origin`: Ray origin point $(O_x, O_y, O_z)$.
- `ray.dir`: Ray direction vector $(D_x, D_y, D_z)$. Must be normalized or consistent with `inv_dir`.
- `ray.inv_dir`: Inverse direction vector $(1/D_x, 1/D_y, 1/D_z)$.
- `aabb.min`: Minimum corner of the grid's axis-aligned bounding box $(A_{min,x}, A_{min,y}, A_{min,z})$.
- `aabb.max`: Maximum corner of the grid's axis-aligned bounding box $(A_{max,x}, A_{max,y}, A_{max,z})$.
- `res`: Grid resolution $(R_x, R_y, R_z)$, representing the number of voxels along each axis.
- `tmin`: Near plane distance along the ray.
- `tmax`: Far plane distance along the ray.
- `eps`: Small epsilon value for numerical stability.

**Outputs:**
- `voxel_size`: Size of a single voxel $(V_x, V_y, V_z)$.
- `current_index`: Starting voxel index $(I_x, I_y, I_z)$.
- `final_index`: Ending voxel index $(F_x, F_y, F_z)$.
- `index_delta`: Step direction per axis $(\Delta I_x, \Delta I_y, \Delta I_z)$.
- `tdist`: Distance to the next voxel boundary along each axis $(T_x, T_y, T_z)$.
- `delta`: Distance step between adjacent voxel boundaries along each axis $(\Delta T_x, \Delta T_y, \Delta T_z)$.

**Data Structures:**
- Vectors are 3-component floats (`float3`) or integers (`int3`).
- `voxel_size`, `ray_start`, `ray_end`, `tdist`, `delta`, `tmax_xyz`, `step_float` are `float3`.
- `current_index`, `final_index`, `index_delta`, `step_index` are `int3`.

**Steps:**

1.  **Calculate Voxel Size:**
    Compute the physical size of a single voxel by dividing the bounding box dimensions by the resolution.
    $$ V = \frac{A_{max} - A_{min}}{R} $$
    Where $V = (V_x, V_y, V_z)$, $A_{max} = (A_{max,x}, A_{max,y}, A_{max,z})$, $A_{min} = (A_{min,x}, A_{min,y}, A_{min,z})$, and $R = (R_x, R_y, R_z)$.

2.  **Compute Ray Start and End Points:**
    Calculate the start and end points of the ray segment within the traversal range, applying epsilon to avoid boundary issues.
    $$ O_{start} = O + D \cdot (t_{min} + \epsilon) $$
    $$ O_{end} = O + D \cdot (t_{max} - \epsilon) $$
    Where $O_{start} = (O_{start,x}, O_{start,y}, O_{start,z})$ and $O_{end} = (O_{end,x}, O_{end,y}, O_{end,z})$.

3.  **Calculate Current and Final Indices:**
    Map the start and end points to voxel indices. This involves "contraction" (clamping the point to the AABB bounds) and scaling by resolution.
    $$ I_{curr} = \text{clamp}\left( \frac{\text{clamp}(O_{start}, A_{min}, A_{max}) - A_{min}}{V} \right) $$
    $$ I_{final} = \text{clamp}\left( \frac{\text{clamp}(O_{end}, A_{min}, A_{max}) - A_{min}}{V} \right) $$
    The result is clamped to $[0, R-1]$ for each component. Note: Some implementations compute this as `apply_contraction(point, aabb.min, aabb.max) * res` then clamp [S2, unofficial].

4.  **Determine Step Direction (`index_delta`):**
    Determine the direction of traversal for each axis based on the ray direction.
    $$ \Delta I_x = \begin{cases} 1 & \text{if } D_x > 0 \\ 0 & \text{otherwise} \end{cases} $$
    $$ \Delta I_y = \begin{cases} 1 & \text{if } D_y > 0 \\ 0 & \text{otherwise} \end{cases} $$
    $$ \Delta I_z = \begin{cases} 1 & \text{if } D_z > 0 \\ 0 & \text{otherwise} \end{cases} $$
    $$ \Delta I = (\Delta I_x, \Delta I_y, \Delta I_z) $$

5.  **Calculate Distance to Next Boundary (`tmax_xyz`):**
    Compute the distance along the ray to the next voxel boundary for each axis. This uses the `start_index` which is `current_index + index_delta` [S1, S2, unofficial].
    $$ I_{start} = I_{curr} + \Delta I $$
    $$ T_{xyz} = \left( \frac{A_{min} + (I_{start} \cdot V) - O_{start}}{D} \right) + t_{min} $$
    Component-wise:
    $$ T_{xyz,x} = \frac{A_{min,x} + (I_{start,x} \cdot V_x) - O_{start,x}}{D_x} + t_{min} $$
    (Similarly for y and z). Note: Division by $D$ is performed using `ray.inv_dir` [S2, unofficial].

6.  **Initialize Traversal Distances (`tdist`):**
    Set the initial distance to the next boundary. Handle rays parallel to grid axes (where $D_i = 0$) by capping the distance to `tmax` to prevent division by zero or infinite loops [S1, S2, unofficial].
    $$ T_x = \begin{cases} t_{max} & \text{if } D_x = 0 \\ T_{xyz,x} & \text{otherwise} \end{cases} $$
    $$ T_y = \begin{cases} t_{max} & \text{if } D_y = 0 \\ T_{xyz,y} & \text{otherwise} \end{cases} $$
    $$ T_z = \begin{cases} t_{max} & \text{if } D_z = 0 \\ T_{xyz,z} & \text{otherwise} \end{cases} $$
    $$ T = (T_x, T_y, T_z) $$

7.  **Compute Step Float (`step_float`):**
    Determine the signed step direction for distance calculations.
    $$ S_x = \begin{cases} 0.0 & \text{if } D_x = 0 \\ 1.0 & \text{if } D_x > 0 \\ -1.0 & \text{otherwise} \end{cases} $$
    (Similarly for y and z).
    $$ S = (S_x, S_y, S_z) $$

8.  **Compute Distance Steps (`delta`):**
    Calculate the distance between adjacent voxel boundaries along each axis.
    $$ \Delta T_x = \begin{cases} t_{max} & \text{if } D_x = 0 \\ V_x \cdot \frac{1}{D_x} \cdot S_x & \text{otherwise} \end{cases} $$
    (Similarly for y and z).
    $$ \Delta T = (\Delta T_x, \Delta T_y, \Delta T_z) $$
    Note: Some sources define `delta_temp` as `voxel_size * ray.inv_dir * step_float` and then apply the zero-direction check [S1, unofficial].

**Pitfalls:**
- **Zero Ray Direction Components:** If $D_x = 0$, `inv_dir` is undefined (division by zero). The algorithm must check for this and set `tdist` to `tmax` and `delta` to `tmax` (or 0, depending on implementation, but `tmax` ensures the axis is skipped in the min-distance check) [S2, unofficial].
- **Epsilon Usage:** Using epsilon prevents the ray from starting exactly on a boundary, which can cause ambiguity in voxel selection [S1, S2, unofficial].
- **Index Clamping:** Indices must be clamped to $[0, R-1]$ to avoid out-of-bounds memory access [S1, S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def grid_traversal_setup(ray_origin, ray_dir, ray_inv_dir, aabb_min, aabb_max, res, tmin, tmax, eps):
    # 1. Calculate Voxel Size
    voxel_size = (aabb_max - aabb_min) / res

    # 2. Compute Ray Start and End Points
    ray_start = ray_origin + ray_dir * (tmin + eps)
    ray_end = ray_origin + ray_dir * (tmax - eps)

    # Helper: Contraction (clamp to AABB)
    def contract(point, min_val, max_val):
        return max(min_val, min(point, max_val))

    # 3. Calculate Current and Final Indices
    # Apply contraction, then scale to voxel indices, then clamp to [0, res-1]
    current_index = clamp((contract(ray_start, aabb_min, aabb_max) - aabb_min) / voxel_size, 0, res - 1)
    final_index = clamp((contract(ray_end, aabb_min, aabb_max) - aabb_min) / voxel_size, 0, res - 1)

    # 4. Determine Step Direction (index_delta)
    index_delta = int3(
        1 if ray_dir.x > 0 else 0,
        1 if ray_dir.y > 0 else 0,
        1 if ray_dir.z > 0 else 0
    )

    # 5. Calculate Distance to Next Boundary (tmax_xyz)
    # start_index is current_index + index_delta
    start_index = current_index + index_delta
    # tmax_xyz = ((aabb.min + (start_index * voxel_size) - ray_start) * ray.inv_dir) + tmin
    tmax_xyz = ((aabb_min + (start_index * voxel_size) - ray_start) * ray_inv_dir) + tmin

    # 6. Initialize Traversal Distances (tdist)
    tdist = float3(
        tmax if ray_dir.x == 0.0 else tmax_xyz.x,
        tmax if ray_dir.y == 0.0 else tmax_xyz.y,
        tmax if ray_dir.z == 0.0 else tmax_xyz.z
    )

    # 7. Compute Step Float (step_float)
    step_float = float3(
        0.0 if ray_dir.x == 0.0 else (1.0 if ray_dir.x > 0.0 else -1.0),
        0.0 if ray_dir.y == 0.0 else (1.0 if ray_dir.y > 0.0 else -1.0),
        0.0 if ray_dir.z == 0.0 else (1.0 if ray_dir.z > 0.0 else -1.0)
    )

    # 8. Compute Distance Steps (delta)
    # delta_temp = voxel_size * ray.inv_dir * step_float
    delta_temp = voxel_size * ray_inv_dir * step_float
    delta = float3(
        tmax if ray_dir.x == 0.0 else delta_temp.x,
        tmax if ray_dir.y == 0.0 else delta_temp.y,
        tmax if ray_dir.z == 0.0 else delta_temp.z
    )

    return voxel_size, current_index, final_index, index_delta, tdist, delta
```

## Subtopics

### [Binary Grid Ray Traversal Initialization](12500-binary-grid-ray-traversal-initialization.md)
Binary Grid Ray Traversal Initialization computes the initial distances to the next voxel boundaries (`tdist`) and the distance steps between adjacent voxels (`delta`) for a ray traversing a binary occupancy grid. It handles rays parallel to grid axes by clamping distances to `tmax` and setting steps to zero, ensuring robust traversal without division by zero. It differs from continuous volume rendering methods by operating on discrete binary occupancy, making it suitable for sparse or binary voxel representations where only presence/absence matters.

## Operating characteristics
**Time Complexity:** $O(1)$ per ray. The setup involves a constant number of arithmetic operations and comparisons.
**Memory Complexity:** $O(1)$ per ray. It stores a fixed set of scalar and vector variables.
**Measured Performance:** Performance depends on the hardware's ability to perform vectorized arithmetic. The method is highly efficient and is typically the bottleneck only in terms of instruction throughput for very high-resolution grids.
**Scaling:** Scales linearly with the number of rays, as each ray's setup is independent.
**Strengths:** Robust handling of edge cases (parallel rays, boundary conditions); simple arithmetic; no branching inside the main traversal loop (branching is limited to setup).
**Limitations:** Requires careful handling of zero-direction components to avoid division by zero. Precision issues may arise with very large or very small voxel sizes if not using appropriate data types.
**When to Use:** When traversing uniform voxel grids for volume rendering or ray marching.
**When Not to Use:** For non-uniform grids (use adaptive methods) or when pre-computing all samples is feasible and memory is not a constraint (use one-pass over-allocated).

## Background
The Digital Differential Analyzer (DDA) for grid traversal was introduced by Amanatides and Woo in 1987 [S1, S2, unofficial]. It is a standard algorithm in computer graphics for efficiently traversing voxel grids. The method is widely used in volume rendering, ray tracing, and spatial partitioning.

## Sources
[S1] taylor2nerf/taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
[S2] YihangChen-ee/CNC, https://github.com/YihangChen-ee/CNC, fetched 2026-10-27 (unofficial)
