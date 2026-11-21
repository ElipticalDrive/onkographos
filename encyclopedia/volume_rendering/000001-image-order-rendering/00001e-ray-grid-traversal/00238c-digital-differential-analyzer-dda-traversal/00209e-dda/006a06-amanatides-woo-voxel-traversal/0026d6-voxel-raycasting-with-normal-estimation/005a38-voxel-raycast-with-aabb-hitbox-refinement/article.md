# Voxel Raycast with AABB Hitbox Refinement

Also called: Voxel Raycasting with AABB Refinement

## Overview
This method extends basic voxel raycasting by combining axis-aligned grid traversal with precise geometric intersection against sub-voxel bounding boxes. It solves the problem of accurately determining surface contact points and normals in voxel environments where blocks may have non-cuboid collision shapes (hitboxes). Unlike its parent method, which returns only the voxel face normal for solid blocks, this algorithm refines the hit position and normal by intersecting the ray against specific AABB hitboxes for non-solid voxels, or falls back to the grid face normal for solid voxels.

## How it works
### Inputs and Outputs
**Inputs:**
*   Ray origin: $(p_x, p_y, p_z)$
*   Ray direction: $(d_x, d_y, d_z)$
*   Maximum traversal distance: $t_{max}$
*   Voxel grid data structure providing access to voxel definitions by integer coordinates.
*   Configuration flags: `includeNonSelectable` (boolean), `excludeMode` (filter logic).

**Outputs:**
*   Hit voxel coordinates: $(i_x, i_y, i_z)$
*   Hit distance: $t$
*   Hit normal: $\mathbf{n} = (n_x, n_y, n_z)$
*   Hit position: $\mathbf{p}_{hit} = \mathbf{p}_{origin} + \mathbf{n}_{dir} \cdot t$ (where $\mathbf{n}_{dir}$ is the normalized ray direction).
*   Null/None if no hit is found within $t_{max}$.

### Data Structures
*   **Voxel Definition:** Contains at minimum:
    *   `id`: Integer identifier.
    *   `solid`: Boolean indicating if the voxel is fully solid.
    *   `selectable`: Boolean indicating if the voxel can be targeted.
    *   `hitboxes`: List of AABBs. Each AABB is defined by min/max coordinates relative to the voxel origin: $(min_x, min_y, min_z)$ and $(max_x, max_y, max_z)$.
    *   `segment_state`: Optional integer or enum used to offset hitbox positions.

### Parameters
*   `epsilon`: Threshold for zero-direction detection. Value: $10^{-6}$.

### Algorithm Steps

1.  **Initialization**:
    *   Calculate initial voxel coordinates:
        $$i_x = \lfloor p_x \rfloor, \quad i_y = \lfloor p_y \rfloor, \quad i_z = \lfloor p_z \rfloor$$
    *   Determine step directions for each axis:
        $$s_x = \text{sign}(d_x), \quad s_y = \text{sign}(d_y), \quad s_z = \text{sign}(d_z)$$
        Where $\text{sign}(v)$ is $1$ if $v \ge 0$ and $-1$ if $v < 0$.
    *   Calculate delta distances to cross the next voxel boundary:
        $$t_{xDelta} = \begin{cases} \infty & \text{if } |d_x| < \epsilon \\ \frac{1.0}{|d_x|} & \text{otherwise} \end{cases}$$
        $$t_{yDelta} = \begin{cases} \infty & \text{if } |d_y| < \epsilon \\ \frac{1.0}{|d_y|} & \text{otherwise} \end{cases}$$
        $$t_{zDelta} = \begin{cases} \infty & \text{if } |d_z| < \epsilon \\ \frac{1.0}{|d_z|} & \text{otherwise} \end{cases}$$
    *   Calculate initial distance to the first boundary ($t_{xMax}, t_{yMax}, t_{zMax}$):
        If $d_x \ge 0$: $x_{dist} = (i_x + 1) - p_x$
        If $d_x < 0$: $x_{dist} = p_x - i_x$
        $$t_{xMax} = t_{xDelta} \cdot x_{dist}$$
        Repeat similarly for $y$ and $z$ axes.
    *   Initialize current distance $t = 0$.
    *   Initialize `steppedIndex` to -1 (indicating no axis stepped yet).

2.  **Traversal Loop**:
    While $t \le t_{max}$:
    *   Retrieve voxel definition at $(i_x, i_y, i_z)$. If voxel is null or air, return null.
    *   **Selectability Check**:
        If `voxel.id` is not `BLOCK_AIR` AND (`voxel.selectable` is true OR `includeNonSelectable` is true) AND (`voxel.id` passes filter based on `excludeMode`):
        *   Proceed to hit detection.

    *   **Hit Detection**:
        *   **Case A: Non-Solid Voxel** (`voxel.solid` is false):
            *   Initialize `closest_t` = $\infty$, `closest_normal` = $(0,0,0)$.
            *   For each hitbox in `voxel.hitboxes`:
                *   Determine AABB bounds. If `voxel.segment_state` exists, apply offset to hitbox coordinates.
                *   Perform Ray-AABB intersection test between ray $(\mathbf{p}_{origin}, \mathbf{d}_{dir})$ and AABB.
                *   If intersection exists at distance $t_{hit} < t_{max}$ and $t_{hit} < closest_t$:
                    *   $closest_t = t_{hit}$
                    *   $closest_normal = \text{intersection normal}$
            *   If `closest_t` < $\infty$:
                *   Return hit: voxel, $t = closest_t$, $\mathbf{n} = closest_normal$.
                *   Update end position: $\mathbf{p}_{end} = \mathbf{p}_{origin} + \mathbf{d}_{dir} \cdot t$.

        *   **Case B: Solid Voxel** (`voxel.solid` is true):
            *   Determine the face normal based on the last axis stepped into.
            *   If `steppedIndex` $\ge 0$:
                *   $\mathbf{n} = (0, 0, 0)$
                *   $\mathbf{n}_{steppedIndex} = -s_{steppedIndex}$
            *   Else:
                *   $\mathbf{n} = (0, 0, 0)$ (or handle degenerate start case).
            *   Return hit: voxel, $t$, $\mathbf{n}$.

    *   **Advance Ray**:
        *   Identify the smallest boundary distance:
            $$t_{next} = \min(t_{xMax}, t_{yMax}, t_{zMax})$$
        *   Update current distance: $t = t_{next}$.
        *   Increment coordinate for the axis corresponding to the minimum:
            *   If $t_{xMax}$ is min: $i_x \leftarrow i_x + s_x$, $t_{xMax} \leftarrow t_{xMax} + t_{xDelta}$, `steppedIndex` = 0.
            *   If $t_{yMax}$ is min: $i_y \leftarrow i_y + s_y$, $t_{yMax} \leftarrow t_{yMax} + t_{yDelta}$, `steppedIndex` = 1.
            *   If $t_{zMax}$ is min: $i_z \leftarrow i_z + s_z$, $t_{zMax} \leftarrow t_{zMax} + t_{zDelta}$, `steppedIndex` = 2.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def raycast_voxel_aabb(p_origin, d_dir, t_max, voxel_grid, include_non_selectable=False, exclude_mode=None):
    """
    Performs voxel raycasting with AABB hitbox refinement.
    
    Args:
        p_origin: Tuple (px, py, pz)
        d_dir: Tuple (dx, dy, dz)
        t_max: Float
        voxel_grid: Object with method get_voxel(ix, iy, iz)
        include_non_selectable: Boolean
        exclude_mode: Function or filter logic
        
    Returns:
        Tuple (voxel, t, normal) or None
    """
    epsilon = 1e-6
    
    # 1. Initialization
    ix = math.floor(p_origin[0])
    iy = math.floor(p_origin[1])
    iz = math.floor(p_origin[2])
    
    dx, dy, dz = d_dir
    sx = 1 if dx >= 0 else -1
    sy = 1 if dy >= 0 else -1
    sz = 1 if dz >= 0 else -1
    
    # Delta distances
    t_x_delta = math.inf if abs(dx) < epsilon else 1.0 / abs(dx)
    t_y_delta = math.inf if abs(dy) < epsilon else 1.0 / abs(dy)
    t_z_delta = math.inf if abs(dz) < epsilon else 1.0 / abs(dz)
    
    # Initial max distances
    if dx >= 0:
        x_dist = (ix + 1) - p_origin[0]
    else:
        x_dist = p_origin[0] - ix
    t_x_max = t_x_delta * x_dist
    
    if dy >= 0:
        y_dist = (iy + 1) - p_origin[1]
    else:
        y_dist = p_origin[1] - iy
    t_y_max = t_y_delta * y_dist
    
    if dz >= 0:
        z_dist = (iz + 1) - p_origin[2]
    else:
        z_dist = p_origin[2] - iz
    t_z_max = t_z_delta * z_dist
    
    t = 0.0
    stepped_index = -1
    
    # 2. Traversal Loop
    while t <= t_max:
        voxel = voxel_grid.get_voxel(ix, iy, iz)
        if voxel is None:
            return None
            
        # Selectability Check
        is_air = (voxel.id == "BLOCK_AIR")
        if not is_air:
            is_selectable = voxel.selectable or include_non_selectable
            passes_filter = True # Assume passes if exclude_mode is None or logic handled
            if exclude_mode is not None:
                passes_filter = exclude_mode(voxel.id)
            
            if is_selectable and passes_filter:
                # Hit Detection
                if not voxel.solid:
                    # Non-solid: Check AABBs
                    closest_t = math.inf
                    closest_normal = (0.0, 0.0, 0.0)
                    
                    for hitbox in voxel.hitboxes:
                        # Apply segment state offset if present
                        offset = voxel.get_segment_offset() if hasattr(voxel, 'get_segment_offset') else (0,0,0)
                        min_b = (hitbox.min_x + offset[0], hitbox.min_y + offset[1], hitbox.min_z + offset[2])
                        max_b = (hitbox.max_x + offset[0], hitbox.max_y + offset[1], hitbox.max_z + offset[2])
                        
                        # Ray-AABB Intersection
                        hit_t, hit_n = intersect_ray_aabb(p_origin, d_dir, min_b, max_b)
                        
                        if hit_t is not None and hit_t < t_max and hit_t < closest_t:
                            closest_t = hit_t
                            closest_normal = hit_n
                    
                    if closest_t < math.inf:
                        # Return hit
                        return (voxel, closest_t, closest_normal)
                else:
                    # Solid: Return face normal
                    normal = [0.0, 0.0, 0.0]
                    if stepped_index >= 0:
                        step_dirs = [sx, sy, sz]
                        normal[stepped_index] = -step_dirs[stepped_index]
                    return (voxel, t, tuple(normal))
        
        # Advance Ray
        if t_x_max < t_y_max:
            if t_x_max < t_z_max:
                t = t_x_max
                ix += sx
                t_x_max += t_x_delta
                stepped_index = 0
            else:
                t = t_z_max
                iz += sz
                t_z_max += t_z_delta
                stepped_index = 2
        else:
            if t_y_max < t_z_max:
                t = t_y_max
                iy += sy
                t_y_max += t_y_delta
                stepped_index = 1
            else:
                t = t_z_max
                iz += sz
                t_z_max += t_z_delta
                stepped_index = 2

    return None

def intersect_ray_aabb(origin, direction, min_b, max_b):
    """
    Standard Ray-AABB intersection. Returns (t, normal) or (None, None).
    """
    # Initialize t_min and t_max
    t_min = -math.inf
    t_max = math.inf
    
    # Precompute inverse direction
    inv_dir = [1.0/d if abs(d) > 1e-9 else 0.0 for d in direction]
    
    normals = []
    
    for i in range(3):
        if inv_dir[i] >= 0:
            t1 = (min_b[i] - origin[i]) * inv_dir[i]
            t2 = (max_b[i] - origin[i]) * inv_dir[i]
            n = [-1.0 if j == i else 0.0 for j in range(3)]
        else:
            t1 = (max_b[i] - origin[i]) * inv_dir[i]
            t2 = (min_b[i] - origin[i]) * inv_dir[i]
            n = [1.0 if j == i else 0.0 for j in range(3)]
            
        if t1 > t_min:
            t_min = t1
            normals = [n]
        elif abs(t1 - t_min) < 1e-9:
            normals.append(n)
            
        if t2 < t_max:
            t_max = t2
        else:
            return None, None
            
    if t_min > t_max:
        return None, None
        
    if t_min < 0:
        # Ray starts inside or behind
        # For this algorithm, we usually want the first positive hit
        # However, standard DDA assumes entering from outside.
        # If t_min < 0, check t_max.
        if t_max < 0:
            return None, None
        # If inside, return the exit normal? Or ignore?
        # The source implies finding the "closest hit distance".
        # If the ray starts inside a hitbox, t_min is negative.
        # We should likely return the positive intersection if it exists.
        if t_max > 0:
             # Return exit normal
             # Re-calculate normal for t_max
             # Simplified: return None for complex inside cases unless specified
             # Given the context of voxel traversal, we usually hit from outside.
             return None, None
             
    # Return normal corresponding to t_min
    # If multiple normals have same t_min, pick one (e.g., first)
    if normals:
        return t_min, normals[0]
    return None, None
```

### Edge Cases and Pitfalls
*   **Division by Zero**: If a ray direction component is effectively zero ($|d_i| < \epsilon$), the delta distance is set to infinity. This ensures the axis is never chosen for stepping, preventing division by zero and infinite loops [S1, unofficial].
*   **Degenerate Rays**: If all direction components are zero, the ray does not move. The algorithm should handle this by returning null or handling it before traversal.
*   **Precision**: Floating-point precision errors in boundary calculations can cause the ray to skip voxels or re-enter the same voxel. Using epsilon for comparisons helps mitigate this.
*   **Hitbox Offsets**: If a voxel has a `segment_state`, hitbox coordinates must be adjusted by the offset before intersection testing. Failure to do so results in incorrect hit positions [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N)$ where $N$ is the number of voxels traversed along the ray. For each non-solid voxel hit, an additional $O(K)$ cost is incurred where $K$ is the number of hitboxes in that voxel.
*   **Memory Complexity**: $O(1)$ auxiliary space, excluding the voxel grid storage.
*   **Strengths**: Provides accurate surface normals and hit positions for non-cuboid collision shapes within a voxel grid. Combines the speed of grid traversal with the precision of geometric intersection.
*   **Limitations**: More computationally expensive than simple face-normal raycasting due to AABB intersection tests. Performance degrades if voxels have many hitboxes.
*   **When to Use**: When voxel blocks have complex collision shapes (e.g., stairs, slabs, custom models) and accurate normal estimation is required for physics or rendering.
*   **When Not to Use**: For simple cubic voxel worlds where face normals are sufficient, or when performance is critical and hitbox precision is not needed.

## Background
This method is an extension of voxel raycasting techniques used in voxel-based games and simulations. It builds upon the Amanatides-Woo voxel traversal algorithm [Parent](3901-voxel-raycasting-with-normal-estimation.md) by adding a layer of geometric precision for non-solid blocks. The specific implementation details, including the handling of segment states and hitbox offsets, are derived from the `voxelcore` library [S1, unofficial].

## Sources
[S1] MihailRis/voxelcore, https://github.com/MihailRis/voxelcore, fetched 2026-10-26 (unofficial)
