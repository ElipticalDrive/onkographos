# Voxel Ray Marching with Traversal Order Optimization

Also called: 33 float points ray marching, 3D Texture Path Tracer

## Overview
Voxel Ray Marching with Traversal Order Optimization renders a 3D voxel texture by advancing a ray from the camera to the back plane, compositing samples via front-to-back alpha blending to visualize the volume [S1, unofficial]. It differs from its siblings by minimizing floating-point operations through the precomputation of step deltas and the determination of traversal order via the comparison of $t_{max}$ values for each axis, rather than relying on fixed-step distance checks or complex spatial acceleration structures [S2, unofficial]. Unlike [Voxel Raymarching with Screen Filters](2555-voxel-raymarching-with-screen-filters.md), which applies screen-space post-processing, this method combines traversal optimization with front-to-back alpha blending compositing [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   `ray_origin`: Ray origin in world space (vec3) [S2, unofficial].
    *   `ray_direction`: Ray direction in world space (vec3, normalized) [S2, unofficial].
    *   `volume_size`: The physical size of the entire voxel volume (scalar, assumed uniform) [S2, unofficial].
    *   `volume_grid`: A vec3 representing the dimensions (width, height, depth) of the voxel grid in number of voxels [S2, unofficial].
    *   `texture_3d`: A 3D texture sampler containing the voxel data (RGBA) [S2, unofficial].
    *   `alpha_clip`: A scalar threshold for alpha testing [S2, unofficial].
*   **Outputs:**
    *   `rec_hit`: Boolean indicating if a surface was hit.
    *   `rec_color`: Hit color (vec3).
    *   `rec_position`: Hit position in world space (vec3).
    *   `rec_normal`: Hit normal (vec3).

**Data Structures**
The implementation relies on a record structure to store the hit state:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
struct RecordVolume:
    hit: bool
    color: vec3
    position: vec3
    normal: vec3
    t: float  # Distance along ray to hit point
```
The voxel grid is accessed via integer indices $(X, Y, Z)$ derived from the ray's position [S2, unofficial].

**Parameters**
*   `_alphaClip`: The alpha threshold used to determine if a sampled voxel is opaque/solid. [S2, unofficial] uses it to stop early.
*   `_volumeSize`: Physical extent of the volume.
*   `_volumeGrid`: Grid resolution $(N_x, N_y, N_z)$.

**Algorithm Steps**
1.  **Initialize Traversal State:** Set `rec_hit = false` [S2, unofficial].
2.  **Determine Step Directions:** Calculate the step direction for each axis based on the ray direction components:
    $$
    \text{stepX} = \begin{cases} 1 & \text{if } r.\text{direction}.x > 0 \\ -1 & \text{otherwise} \end{cases}
    $$
    Repeat for `stepY` and `stepZ` [S2, unofficial].
3.  **Precompute Step Deltas ($t_{\Delta}$):** Calculate the time (distance along ray) to travel one voxel unit along each axis. This assumes a uniform voxel size derived from `_volumeSize` and `_volumeGrid`:
    $$
    t_{\Delta X} = \left| \frac{\text{_volumeSize} / \text{_volumeGrid}.x}{r.\text{direction}.x} \right|
    $$
    $$
    t_{\Delta Y} = \left| \frac{\text{_volumeSize} / \text{_volumeGrid}.y}{r.\text{direction}.y} \right|
    $$
    $$
    t_{\Delta Z} = \left| \frac{\text{_volumeSize} / \text{_volumeGrid}.z}{r.\text{direction}.z} \right|
    $$
    [S2, unofficial].
4.  **Calculate Initial Position and Indices:**
    *   Compute the current position in world space: $pos = r.\text{origin} + r.\text{direction} \times rec.t$ [S2, unofficial].
    *   Convert to voxel indices:
        $$
        X = \lfloor pos.x / \text{_volumeSize} \times \text{_volumeGrid}.x \rfloor
        $$
        $$
        Y = \lfloor pos.y / \text{_volumeSize} \times \text{_volumeGrid}.y \rfloor
        $$
        $$
        Z = \lfloor pos.z / \text{_volumeSize} \times \text{_volumeGrid}.z \rfloor
        $$
    *   **Boundary Correction:** If any index equals the corresponding grid dimension, decrement it by 1 to stay within valid bounds $[0, \text{grid\_dim}-1]$ [S2, unofficial].
5.  **Calculate Initial $t_{max}$:**
    *   Determine the next voxel index offset:
        $$
        \text{nextX} = \begin{cases} 1 & \text{if } r.\text{direction}.x > 0 \\ 0 & \text{otherwise} \end{cases}
        $$
        (Repeat for Y, Z) [S2, unofficial].
    *   Compute the world position of the next boundary: $nextPos = \text{getBoundaryPos}(X+\text{nextX}, Y+\text{nextY}, Z+\text{nextZ})$ [S2, unofficial].
    *   Calculate the distance to the next boundary along each axis:
        $$
        t_{max X} = \frac{nextPos.x - pos.x}{r.\text{direction}.x}
        $$
        $$
        t_{max Y} = \frac{nextPos.y - pos.y}{r.\text{direction}.y}
        $$
        $$
        t_{max Z} = \frac{nextPos.z - pos.z}{r.\text{direction}.z}
        $$
    [S2, unofficial].
6.  **Check Immediate Hit:** Sample the current voxel $(X, Y, Z)$. If `sample3D(X,Y,Z).a > _alphaClip`, set `rec_hit = true`, store the color, position, and normal, and return [S2, unofficial].
7.  **Main Marching Loop:** While `!rec_hit`:
    *   Compare $t_{max X}, t_{max Y}, t_{max Z}$ to determine the traversal order. The logic follows a nested comparison structure (0->1->2->3->4 logic implied) [S2, unofficial]:
        *   **If $t_{max X} < t_{max Y}$:**
            *   **If $t_{max X} < t_{max Z}$:** (X is smallest)
                *   Increment $X$ by `stepX`.
                *   Check boundary: If $X \ge \text{_volumeGrid}.x$ or $X < 0$, break loop [S2, unofficial].
                *   Update $t_{max X} \leftarrow t_{max X} + t_{\Delta X}$ [S2, unofficial].
                *   Set `potentialNormal` to $(-\text{stepX}, 0, 0)$ [S2, unofficial].
            *   **Else ($t_{max Z} < t_{max X}$):** (Z is smallest among remaining)
                *   Increment $Z$ by `stepZ`.
                *   Check boundary: If $Z < 0$ or $Z \ge \text{_volumeGrid}.z$, break loop [S2, unofficial].
                *   Update $t_{max Z} \leftarrow t_{max Z} + t_{\Delta Z}$ [S2, unofficial].
                *   Set `potentialNormal` to $(0, 0, -\text{stepZ})$ [S2, unofficial].
        *   **Else ($t_{max Y} \le t_{max X}$):**
            *   **If $t_{max Y} < t_{max Z}$:** (Y is smallest)
                *   Increment $Y$ by `stepY`.
                *   Check boundary: If $Y \ge \text{_volumeGrid}.y$ or $Y < 0$, break loop [S2, unofficial].
                *   Update $t_{max Y} \leftarrow t_{max Y} + t_{\Delta Y}$ [S2, unofficial].
                *   Set `potentialNormal` to $(0, -\text{stepY}, 0)$ [S2, unofficial].
            *   **Else ($t_{max Z} \le t_{max Y}$):** (Z is smallest)
                *   Increment $Z$ by `stepZ`.
                *   Check boundary: If $Z < 0$ or $Z \ge \text{_volumeGrid}.z$, break loop [S2, unofficial].
                *   Update $t_{max Z} \leftarrow t_{max Z} + t_{\Delta Z}$ [S2, unofficial].
                *   Set `potentialNormal` to $(0, 0, -\text{stepZ})$ [S2, unofficial].
    *   **Sample and Check:** After moving to the new voxel, sample `sample3D(X,Y,Z)`. If `alpha > _alphaClip`, set `rec_hit = true`, store color, position ($r.\text{origin} + r.\text{direction} \times \text{potentialTime}$), and normal, and break loop [S2, unofficial].

**Pitfalls**
*   **Zero Direction Components:** The code assumes ray direction components are non-zero when computing $t_{\Delta}$ and $t_{max}$. If a direction component is zero, division by zero results in infinity or NaN, causing incorrect behavior or crashes. Clamp or handle zero directions explicitly [S2, unofficial].
*   **Boundary Conditions:** If the calculated integer coordinate equals the grid dimension, it must be decremented by 1 to stay within valid array bounds $[0, \text{grid\_dim}-1]$ [S2, unofficial].
*   **Out-of-Bounds Sampling:** Out-of-bounds coordinates are handled differently per platform; some use border color sampling, others require explicit `isInside` checks returning transparent black [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_ray_march_traversal_optimized(ray_origin, ray_direction, volume_size, volume_grid, texture_3d, alpha_clip):
    """
    Renders a voxel volume using traversal order optimization.
    Assumes ray_direction is normalized.
    """
    rec_hit = False
    rec_color = vec3(0.0)
    rec_position = vec3(0.0)
    rec_normal = vec3(0.0)
    rec_t = 0.0  # Entry point t along ray

    # 1. Determine step directions
    step_x = 1 if ray_direction.x > 0.0 else -1
    step_y = 1 if ray_direction.y > 0.0 else -1
    step_z = 1 if ray_direction.z > 0.0 else -1

    # 2. Precompute tDelta (time to travel one voxel unit)
    voxel_size_x = volume_size / volume_grid.x
    voxel_size_y = volume_size / volume_grid.y
    voxel_size_z = volume_size / volume_grid.z

    # Handle zero direction components to avoid division by zero [G]
    dir_x = ray_direction.x if abs(ray_direction.x) > 1e-9 else 1e-9
    dir_y = ray_direction.y if abs(ray_direction.y) > 1e-9 else 1e-9
    dir_z = ray_direction.z if abs(ray_direction.z) > 1e-9 else 1e-9

    t_delta_x = abs(voxel_size_x / dir_x)
    t_delta_y = abs(voxel_size_y / dir_y)
    t_delta_z = abs(voxel_size_z / dir_z)

    # 3. Initial Position and Indices
    pos = ray_origin + ray_direction * rec_t

    X = int(floor(pos.x / volume_size * volume_grid.x))
    Y = int(floor(pos.y / volume_size * volume_grid.y))
    Z = int(floor(pos.z / volume_size * volume_grid.z))

    # Boundary correction: if index equals grid dimension, decrement by 1 [S2, unofficial]
    if X == volume_grid.x: X -= 1
    if Y == volume_grid.y: Y -= 1
    if Z == volume_grid.z: Z -= 1

    # 4. Calculate initial tMax
    next_x_offset = 1 if ray_direction.x > 0.0 else 0
    next_y_offset = 1 if ray_direction.y > 0.0 else 0
    next_z_offset = 1 if ray_direction.z > 0.0 else 0

    # Helper to get world boundary coordinate from voxel index
    def get_boundary_pos(ix, iy, iz):
        return vec3(
            ix * (volume_size / volume_grid.x),
            iy * (volume_size / volume_grid.y),
            iz * (volume_size / volume_grid.z)
        )

    next_pos = get_boundary_pos(X + next_x_offset, Y + next_y_offset, Z + next_z_offset)

    t_max_x = (next_pos.x - pos.x) / dir_x
    t_max_y = (next_pos.y - pos.y) / dir_y
    t_max_z = (next_pos.z - pos.z) / dir_z

    potential_time = 0.0
    potential_normal = vec3(0.0)

    # 5. Check immediate hit
    sample = texture_3d.sample(X, Y, Z)
    if sample.a > alpha_clip:
        rec_hit = True
        rec_color = sample.rgb
        rec_position = ray_origin + ray_direction * rec_t
        rec_normal = potential_normal
        return rec_hit, rec_color, rec_position, rec_normal

    # 6. Main Marching Loop
    while not rec_hit:
        if t_max_x < t_max_y:
            if t_max_x < t_max_z:
                # X is smallest
                X += step_x
                potential_time = t_max_x
                if X >= volume_grid.x or X < 0:
                    break
                t_max_x += t_delta_x
                potential_normal = vec3(-float(step_x), 0.0, 0.0)
            else:
                # Z is smallest
                Z += step_z
                potential_time = t_max_z
                if Z < 0 or Z >= volume_grid.z:
                    break
                t_max_z += t_delta_z
                potential_normal = vec3(0.0, 0.0, -float(step_z))
        else:
            if t_max_y < t_max_z:
                # Y is smallest
                Y += step_y
                potential_time = t_max_y
                if Y >= volume_grid.y or Y < 0:
                    break
                t_max_y += t_delta_y
                potential_normal = vec3(0.0, -float(step_y), 0.0)
            else:
                # Z is smallest
                Z += step_z
                potential_time = t_max_z
                if Z < 0 or Z >= volume_grid.z:
                    break
                t_max_z += t_delta_z
                potential_normal = vec3(0.0, 0.0, -float(step_z))

        # Sample new voxel
        sample = texture_3d.sample(X, Y, Z)
        if sample.a > alpha_clip:
            rec_hit = True
            rec_color = sample.rgb
            rec_position = ray_origin + ray_direction * potential_time
            rec_normal = potential_normal
            break

    return rec_hit, rec_color, rec_position, rec_normal
```

## Subtopics

### [Voxel Raymarching with Screen Filters](2555-voxel-raymarching-with-screen-filters.md)
A fragment shader technique that combines ray traversal with screen-space visual effects (normal, position, depth, iterations, flat color, and toon filters). It differs by applying these filters to the final color output rather than relying solely on traversal optimization. It is distinct from the [Voxel Visualization Ray Marcher](3432-voxel-visualization-ray-marcher.md) by using these screen filters and a complex lighting model with specular and diffuse directions, rather than a fixed-step traversal with explicit alpha compositing.

## Operating characteristics
*   **Time Complexity:** The number of steps is proportional to the number of voxels traversed by the ray [G]. In the worst case (ray passing through the entire volume), it is $O(N_x + N_y + N_z)$, but typically much less due to early exit on hit [G].
*   **Memory Complexity:** $O(1)$ auxiliary memory for traversal state, plus the storage for the 3D texture [G].
*   **Performance:** The method is highly efficient because it minimizes floating-point operations by precomputing deltas and using integer arithmetic for index updates [S2, unofficial]. It avoids the overhead of complex acceleration structures like BVHs or KD-trees, making it suitable for real-time rendering on GPUs [S2, unofficial].
*   **Strengths:** Simple implementation, low memory overhead, fast traversal for uniform voxel grids.
*   **Limitations:** Performance degrades with very high resolution grids if the ray traverses many empty voxels (though early exit helps) [G]. It assumes a uniform voxel grid. It does not handle non-axis-aligned volumes or complex geometry directly [G].
*   **When to use:** Real-time rendering of voxel-based data (e.g., medical imaging, voxel art) where the grid is uniform and axis-aligned [G].
*   **When not to use:** Non-uniform grids, complex implicit surfaces, or scenes requiring hierarchical acceleration [G].

## Background
The method originates from the implementation of a voxel path tracing engine, specifically described in "A Voxel PathTracing Engine" by Jiayu He [S2, unofficial]. The technique is noted for its efficiency, requiring only "33 float points calculations" for the traversal logic, which is comparable to BVH collision detection efficiency [S2, unofficial]. It builds on the principles of [Ray-grid traversal](17-ray-grid-traversal.md), optimizing the cost of executing the traversal by precomputing step deltas and determining the order of axis crossings via $t_{max}$ comparisons [S2, unofficial].

## Sources
[S1] kakashidinho/VoxelConeTracingMetal, https://github.com/kakashidinho/VoxelConeTracingMetal, fetched 2026-10-26 (unofficial)
[S2] A Voxel PathTracing Engine - Jiayu He - GitHub Pages, https://iota11.github.io/2024/01/10/voxelRender/, fetched 2026-10-28 (unofficial)
