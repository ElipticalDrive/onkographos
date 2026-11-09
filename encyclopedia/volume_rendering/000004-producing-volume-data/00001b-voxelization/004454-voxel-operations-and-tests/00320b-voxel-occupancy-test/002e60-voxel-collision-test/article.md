# Voxel Collision Test

Also called: Grid Check, Voxel Column Collision, vertical column check

## Overview
A Voxel Collision Test determines if a solid voxel exists within a specific spatial region, typically a vertical column defined by world XZ coordinates and a Y-axis slab range. It solves the problem of detecting physical obstructions in voxel-based environments, enabling mechanics such as walking through carved-out holes or detecting ground contact for entity physics. It differs from the parent [Voxel Occupancy Test](1281-voxel-occupancy-test.md), which tests individual points, by aggregating occupancy over a continuous volume (a column or slab) to return a single binary result for the region.

## How it works
The method maps world-space coordinates to local voxel grid indices and iterates through the occupied cells within the defined bounds.

### Inputs and Outputs
*   **Inputs**:
    *   `world_xz`: A 2D position $(x, z)$ in world coordinates.
    *   `slab_lo`, `slab_hi`: The lower and upper Y bounds in world coordinates.
    *   `grid_origin`: The world position of the voxel grid's $(0,0,0)$ corner.
    *   `vox_size`: The side length of a single voxel.
    *   `grid_dim`: The dimensions of the voxel grid $(D_x, D_y, D_z)$.
    *   `grid_data`: A 3D array or sparse structure storing voxel states (e.g., solid/empty).
    *   `rotation`: (Optional) The orientation of the voxel grid relative to the world, if the grid is not axis-aligned with the world axes.
*   **Outputs**:
    *   `collision`: A boolean indicating if any voxel in the column is solid.

### Data Structures
*   **Voxel Grid**: A 3D array `grid[D_x][D_y][D_z]` where each element is a boolean or integer flag. `true` or non-zero indicates a solid voxel; `false` or zero indicates empty space.
*   **Local Coordinates**: Temporary floating-point vectors used to map world positions to grid indices.

### Steps
1.  **Coordinate Transformation**: Convert the world XZ position and slab Y bounds into the voxel grid's local coordinate system. If the grid is axis-aligned with the world, this is a simple translation. If rotated, apply the inverse rotation.
    *   Let $P_{xz} = (world\_xz.x, 0.0, world\_xz.y)$.
    *   Let $P_{local} = R^{-1} \cdot (P_{xz} - grid\_origin)$, where $R$ is the grid's rotation matrix.
    *   Let $y_{local\_lo} = slab\_lo - grid\_origin.y$.
    *   Let $y_{local\_hi} = slab\_hi - grid\_origin.y$.
    *   If rotation is applied, $y_{local}$ components must also be transformed by $R^{-1}$. In the common axis-aligned case, $y_{local\_lo} = slab\_lo - grid\_origin.y$ and $y_{local\_hi} = slab\_hi - grid\_origin.y$.

2.  **Calculate Grid Indices**: Convert local coordinates to floating-point grid indices.
    *   $lx_f = \frac{local.x + half\_grid}{vox\_size}$
    *   $lz_f = \frac{local.z + half\_grid}{vox\_size}$
    *   $ly\_lo_f = \frac{y_{local\_lo} + half\_grid}{vox\_size}$
    *   $ly\_hi_f = \frac{y_{local\_hi} + half\_grid}{vox\_size}$
    *   Where $half\_grid$ is typically $0.5 \times vox\_size$ or $0.0$ depending on whether the grid origin is at the corner or center of the first voxel. The source [S2, unofficial] uses `half_grid` in the numerator, implying a center-based indexing or offset. We define $half\_grid = 0.5 \times vox\_size$ for standard corner-origin grids where index 0 covers $[0, vox\_size)$. *Note: Source [S2, unofficial] explicitly adds `half_grid` to the position before dividing, suggesting the local coordinate system might be centered or the formula accounts for a specific offset convention.*

3.  **Boundary Check (XZ)**: Check if the X and Z indices fall within the grid dimensions.
    *   If $lx_f < 0$ or $lx_f \ge D_x$ or $lz_f < 0$ or $lz_f \ge D_z$, return `false` (no collision, out of bounds).

4.  **Calculate Y Bounds**: Convert floating-point Y indices to integer grid indices.
    *   $ly\_lo = \lfloor ly\_lo_f \rfloor$
    *   $ly\_hi = \lceil ly\_hi_f \rceil$
    *   Clamp $ly\_lo$ to $[0, D_y - 1]$.
    *   Clamp $ly\_hi$ to $[0, D_y - 1]$.
    *   If $ly\_lo > ly\_hi$, return `false` (empty slab).

5.  **Iterate and Test**: Loop through Y indices from $ly\_lo$ to $ly\_hi$.
    *   For each $y$ in $[ly\_lo, ly\_hi]$:
        *   Check `grid[round(lx_f)][y][round(lz_f)]`.
        *   If the voxel is solid, return `true`.

6.  **Return**: If the loop completes without finding a solid voxel, return `false`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxel_collision_test(world_xz, slab_lo, slab_hi, grid_origin, vox_size, grid_dim, grid_data, rotation=None):
    """
    world_xz: tuple (x, z)
    slab_lo, slab_hi: float
    grid_origin: tuple (ox, oy, oz)
    vox_size: float
    grid_dim: tuple (dx, dy, dz)
    grid_data: 3D list or array
    rotation: 3x3 matrix or None for axis-aligned
    """
    # 1. Transform to local coordinates
    # World position vector (Y is 0 for the column base, but we transform the XZ plane)
    # In axis-aligned case, local XZ is just world XZ - origin XZ
    if rotation is None:
        local_x = world_xz[0] - grid_origin[0]
        local_z = world_xz[1] - grid_origin[2]
        local_y_lo = slab_lo - grid_origin[1]
        local_y_hi = slab_hi - grid_origin[1]
    else:
        # Apply inverse rotation to the vector from origin to world_xz (with y=0)
        # Note: Source [S2, unofficial] constructs a Vec3 with world_xz.x, 0.0, world_xz.y
        p_world = (world_xz[0], 0.0, world_xz[1])
        p_local_vec = rotation.inverse() * (p_world - grid_origin)
        local_x = p_local_vec[0]
        local_z = p_local_vec[2]
        # Y transformation for slab bounds requires transforming the Y vector as well
        # Assuming rotation affects Y, we transform the Y offsets
        y_vec_lo = (0.0, slab_lo - grid_origin[1], 0.0)
        y_vec_hi = (0.0, slab_hi - grid_origin[1], 0.0)
        # This part depends on how rotation is applied to Y. 
        # Source [S2, unofficial] only rotates the XZ position for the column lookup, 
        # but transforms Y bounds separately? 
        # Re-reading [S2, unofficial]: "Convert the slab Y range (world coordinates) to local grid Y indices"
        # It does NOT apply rotation to Y in the snippet provided, implying axis-aligned Y or 
        # that the rotation is only around Y axis (yaw). 
        # If rotation is arbitrary, Y must be transformed. 
        # For general implementation, we assume rotation might affect Y.
        # However, standard voxel engines often keep Y axis-aligned.
        # Let's stick to the explicit source [S2, unofficial] logic:
        local_y_lo = slab_lo - grid_origin[1] # Source does not rotate Y bounds
        local_y_hi = slab_hi - grid_origin[1]

    half_grid = 0.5 * vox_size

    # 2. Calculate grid indices
    lx_f = (local_x + half_grid) / vox_size
    lz_f = (local_z + half_grid) / vox_size
    ly_lo_f = (local_y_lo + half_grid) / vox_size
    ly_hi_f = (local_y_hi + half_grid) / vox_size

    # 3. Boundary Check XZ
    dx, dy, dz = grid_dim
    if lx_f < 0 or lx_f >= dx or lz_f < 0 or lz_f >= dz:
        return False

    # 4. Calculate Y bounds
    ly_lo = int(ly_lo_f) # floor
    ly_hi = int(ly_hi_f) # ceil logic: if fractional, go to next int
    
    # Correct ceil for positive numbers
    import math
    ly_hi = math.ceil(ly_hi_f)
    
    # Clamp
    ly_lo = max(0, min(ly_lo, dy - 1))
    ly_hi = max(0, min(ly_hi, dy - 1))

    if ly_lo > ly_hi:
        return False

    # 5. Iterate
    # Round X and Z to nearest integer index
    ix = round(lx_f)
    iz = round(lz_f)
    
    # Ensure ix, iz are within bounds after rounding (though float check above covers most)
    ix = max(0, min(ix, dx - 1))
    iz = max(0, min(iz, dz - 1))

    for y in range(ly_lo, ly_hi + 1):
        if grid_data[ix][y][iz]:
            return True

    return False
```

### Parameters
*   `vox_size`: Typically $1.0$ unit per voxel in world space.
*   `half_grid`: $0.5 \times vox\_size$.

### Edge Cases and Pitfalls
*   **Precision**: Floating-point errors in index calculation can lead to off-by-one errors. Using `floor` and `ceil` correctly handles the slab boundaries.
*   **Out of Bounds**: If the XZ position is outside the grid, the test must return `false` (or handle as a boundary condition) rather than crashing.
*   **Empty Slab**: If `slab_lo` and `slab_hi` are equal or inverted, the loop range is empty.
*   **Rotation**: If the grid is rotated, the Y-axis transformation must be consistent with the XZ transformation. Source [S2, unofficial] implies a Y-axis rotation only (yaw), where Y remains unchanged.

## Subtopics

### Solid Voxel Particle Physics
A variant used for GPU particle systems where collisions are detected against a solid voxelization with normals. It treats concave regions and dynamic deformation seamlessly. Normals are recomputed every frame, and evaluated only when a particle enters contact with a surface. It offers uniform quality in all directions, unlike depth peeling or clipping planes, allowing consistent back integration. [S3, unofficial]

### Entity Collision & Movement
A variant used for entity physics including gravity, ground detection, and axis-aligned movement with wall-sliding. It checks blocks at specific offsets (e.g., below the entity) to determine if the entity is on the ground. If a collision is detected during movement, it attempts to slide along the X or Z axis. It uses a block ID of 255 to represent air. [S4, unofficial]

## Operating characteristics
*   **Time Complexity**: $O(D_y)$ for a full column scan, where $D_y$ is the height of the grid. In practice, it is $O(h)$ where $h$ is the number of voxels in the slab.
*   **Memory Complexity**: $O(1)$ auxiliary space, assuming the grid data is pre-loaded.
*   **Strengths**: Simple to implement, robust for axis-aligned grids, handles concave geometry naturally.
*   **Limitations**: Does not provide collision normals or precise contact points, only binary occupancy. Performance degrades with very tall slabs or high-resolution grids.
*   **When to use**: Ground detection, simple obstacle avoidance, voxel-based physics.
*   **When not to use**: When precise collision normals are required (use raycasting or signed distance fields), or when the grid is extremely sparse and dense (use spatial hashing or BVH).

## Background
The method is a fundamental operation in voxel-based game engines and simulations. It is often referred to as a "grid check" in the context of voxel games like *noa* [S1, unofficial]. It is used in various implementations, from simple 2D heightmap checks to 3D entity physics in engines like *Prometheus Engine* [S2, unofficial] and *VoxelEngine* [S4, unofficial]. It is also applied in GPU particle simulations using solid voxelization [S3, unofficial].

## Sources
[S1] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S2] elyssov/prometheus-engine, https://github.com/elyssov/prometheus-engine, fetched 2026-10-28 (unofficial)
[S3] Single-pass GPU Solid Voxelization and Applications, https://inria.hal.science/inria-00345291, fetched 2026-10-26 (unofficial)
[S4] GitHub Repo for VoxelEngine, https://github.com/TheSlabby/VoxelEngine, fetched 2026-10-29 (unofficial)
