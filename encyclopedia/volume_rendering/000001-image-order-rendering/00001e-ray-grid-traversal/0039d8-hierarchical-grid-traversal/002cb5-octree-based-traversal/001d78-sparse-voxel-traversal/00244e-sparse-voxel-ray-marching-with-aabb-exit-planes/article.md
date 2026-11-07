# Sparse Voxel Ray Marching with AABB Exit Planes

Also called: NAADF voxel traversal, Sparse Voxel Ray Marching with NAADF, voxel DDA with precomputed exit planes, vxray sparse ray march

## Overview
Sparse Voxel Ray Marching with AABB Exit Planes is a ray traversal algorithm for sparse voxel hierarchies that skips empty space by computing the exit plane of the current empty cell using precomputed Axis-Aligned Distance Fields (AADF) [S1, unofficial]. It differs from standard DDA [Voxel Grid Raycasting](2218-voxel-grid-raycasting.md) by avoiding per-voxel checks in empty regions, and from pure octree traversal [Octree Traversal](1587-octree-traversal.md) by using fixed-size hierarchical chunks (chunks, bricks, voxels) with texture-based occupancy masks and distance fields rather than dynamic node pointers [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Ray origin `ray_origin`, ray direction `ray_dir`, grid extent `grid_ext` (uniform), occupancy textures (`chunk_masks`, `brick_masks`, `voxel_masks`), AADF textures (`chunk_aadf`, `brick_aadf`, `voxel_aadf`).
- Output: Packed cell info (voxel coordinate and entry axis) or sentinel `VX_NO_CELL` if the ray misses or exits the grid.

**Data Structures:**
- **Occupancy Masks:** Three `Texture3D<uint>` (`chunk_masks`, `brick_masks`, `voxel_masks`). Each texel is a `uint` bitmask. The bit at index `mask_linear_idx(coord)` indicates occupancy for a block of voxels. `mask_linear_idx(coord) = local.x + local.y * VX_MASK_EXT + local.z * VX_MASK_EXT * VX_MASK_EXT`, where `local = coord & (VX_MASK_EXT - 1)` [S1, unofficial].
- **AADF Textures:** Three `Texture3D<uint>` (`chunk_aadf`, `brick_aadf`, `voxel_aadf`). Each texel is a `uint` packing distances to the nearest occupied voxel in 6 directions (±X, ±Y, ±Z) [S1, unofficial].
- **AADF Packing Layout:** Bits 0-4: -X distance, 5-9: +X distance, 10-14: -Y distance, 15-19: +Y distance, 20-24: -Z distance, 25-29: +Z distance [S1, unofficial].
- **Constants:** `VX_MASK_EXT` (mask block size), `VX_BRICK_EXT` (brick size), `VX_CHUNK_EXT` (chunk size), `VX_NO_CELL` (sentinel) [S1, unofficial].

**Numbered Steps:**
1.  **Initialize Ray Inverse:** Compute `inv_dir = 1.0 / (ray_dir + (float3)(ray_dir == 0.0) * 1e-30)` to handle zero components [S1, unofficial].
2.  **Initial AABB Test:** Perform ray-AABB intersection against the entire grid `[0, grid_ext]`. If no intersection, return `VX_NO_CELL` [S1, unofficial].
    -   Formula: `t0 = (p0 - ray_origin) * inv_ray_dir`, `t1 = (p1 - ray_origin) * inv_ray_dir`, `lo = min(t0, t1)`, `hi = max(t0, t1)`, `tmin = max(max(lo.x, lo.y), max(lo.z, 0.0))`, `tmax = min(min(hi.x, hi.y), hi.z)`. Return `tmin <= tmax` [S1, unofficial].
3.  **Initial Position:** Calculate starting position `start = ray_origin + tmin * ray_dir`. Compute initial voxel coordinate `ipos = int16_t3(floor(start + entry_mask * sign(ray_dir) * 0.5))` and local position `local_pos = start - float3(ipos)`. `entry_mask` determines the entry face to adjust `ipos` [S1, unofficial].
4.  **Determine Initial Axis:** Compute `entry_weights = entry_mask * abs(ray_dir)`. Determine `crossed_axis`: 1 if `entry_weights.y > entry_weights.x`, else 0. Then 2 if `entry_weights.z > max(entry_weights.x, entry_weights.y)`, else unchanged [S1, unofficial].
5.  **Loop:**
    a.  **Bounds Check:** If `any((uint16_t3)ipos >= (uint16_t)grid_ext)`, return `VX_NO_CELL` [S1, unofficial].
    b.  **Hierarchy Check:**
        i.  Chunk level: `cell_ext = VX_BRICK_EXT * VX_CHUNK_EXT`, `coord = ipos / cell_ext`. Check `chunk_occupied(coord)`. If occupied, return `pack_voxel_cell(ipos, crossed_axis)` [S1, unofficial].
        ii. Brick level: `cell_ext = VX_BRICK_EXT`, `coord = ipos / cell_ext`. Check `brick_occupied(coord)`. If occupied, return `pack_voxel_cell(ipos, crossed_axis)` [S1, unofficial].
        iii. Voxel level: `cell_ext = 1`, `coord = ipos`. Check `voxel_occupied(coord)`. If occupied, return `pack_voxel_cell(ipos, crossed_axis)` [S1, unofficial].
    c.  **AADF Skip:** If not occupied, load AADF value for the current level (`aadf = chunk_aadf.Load(...)`, etc.) [S1, unofficial].
    d.  **Exit Plane Calculation:** Compute exit plane coordinate using `aadf_exit_plane(coord, cell_ext, aadf, ray_dir)` [S1, unofficial].
        -   Formula: `shift_x = ray_dir.x < 0.0 ? 0u : 5u`, `shift_y = ray_dir.y < 0.0 ? 10u : 15u`, `shift_z = ray_dir.z < 0.0 ? 20u : 25u`. Extract bounds: `bounds.x = (packed >> shift_x) & 31u`, etc. Calculate `lower = max((coord - bounds) * cell_ext, 0)`, `upper = min((coord + 1 + bounds) * cell_ext, grid_ext)`. Return `ray_dir.x < 0.0 ? lower.x : upper.x` for each component [S1, unofficial].
    e.  **Distance to Exit:** Compute `side_dist = (exit_plane - float3(ipos) - local_pos) * inv_dir`. Handle zero ray direction components by setting `side_dist` component to `3e+38` [S1, unofficial].
    f.  **Find Minimum:** Find `t = min(side_dist.x, min(side_dist.y, side_dist.z))`. Determine crossed axis: `crossed = step(side_dist, t)`. Update `crossed_axis` based on weights of crossed axes [S1, unofficial].
    g.  **Advance:** Compute `advanced = local_pos + max(t, 0.0001) * ray_dir`. Compute new integer position `cell_delta = int16_t3(floor(advanced + crossed * sign(ray_dir) * 0.5))`. Update `ipos += cell_delta` and `local_pos = advanced - float3(cell_delta)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_voxel_ray_march(ray_origin, ray_dir, grid_ext, chunk_masks, brick_masks, voxel_masks, chunk_aadf, brick_aadf, voxel_aadf, VX_MASK_EXT, VX_BRICK_EXT, VX_CHUNK_EXT, VX_NO_CELL):
    # Step 1: Initialize Ray Inverse
    inv_dir = 1.0 / (ray_dir + (ray_dir == 0.0) * 1e-30)

    # Step 2: Initial AABB Test
    p0 = float3(0.0)
    p1 = float3(grid_ext)
    t0 = (p0 - ray_origin) * inv_dir
    t1 = (p1 - ray_origin) * inv_dir
    lo = min(t0, t1)
    hi = max(t0, t1)
    tmin = max(max(lo.x, lo.y), max(lo.z, 0.0))
    tmax = min(min(hi.x, hi.y), hi.z)
    if tmin > tmax:
        return VX_NO_CELL

    # Step 3: Initial Position
    start = ray_origin + tmin * ray_dir
    # entry_mask is determined by which face was entered; for simplicity, assume it's computed externally or derived from tmin relative to faces
    # entry_mask = float3(1.0 if ray_dir.x < 0 else 0.0, 1.0 if ray_dir.y < 0 else 0.0, 1.0 if ray_dir.z < 0 else 0.0)
    # Note: entry_mask logic in source is complex; assuming it's available or computed as:
    entry_mask = float3(1.0 if ray_dir.x < 0 else 0.0, 1.0 if ray_dir.y < 0 else 0.0, 1.0 if ray_dir.z < 0 else 0.0)
    ipos = int16_t3(floor(start + entry_mask * sign(ray_dir) * 0.5))
    local_pos = start - float3(ipos)

    # Step 4: Determine Initial Axis
    entry_weights = entry_mask * abs(ray_dir)
    crossed_axis = 0
    if entry_weights.y > entry_weights.x:
        crossed_axis = 1
    if entry_weights.z > max(entry_weights.x, entry_weights.y):
        crossed_axis = 2

    # Step 5: Loop
    while True:
        # 5a: Bounds Check
        if any(ipos >= grid_ext):
            return VX_NO_CELL

        # 5b: Hierarchy Check
        # Chunk Level
        cell_ext = VX_BRICK_EXT * VX_CHUNK_EXT
        coord = ipos / cell_ext
        if chunk_occupied(coord, chunk_masks, VX_MASK_EXT):
            return pack_voxel_cell(ipos, crossed_axis)

        # Brick Level
        cell_ext = VX_BRICK_EXT
        coord = ipos / cell_ext
        if brick_occupied(coord, brick_masks, VX_MASK_EXT):
            return pack_voxel_cell(ipos, crossed_axis)

        # Voxel Level
        cell_ext = 1
        coord = ipos
        if voxel_occupied(coord, voxel_masks, VX_MASK_EXT):
            return pack_voxel_cell(ipos, crossed_axis)

        # 5c: AADF Skip
        # Determine which AADF texture to load based on the last checked level (voxel level in this case, as higher levels were empty)
        # Note: The source checks chunk, then brick, then voxel. If all are empty, we load the voxel AADF.
        aadf = voxel_aadf.Load(int4(coord, 0))

        # 5d: Exit Plane Calculation
        exit_plane = aadf_exit_plane(coord, cell_ext, aadf, ray_dir)

        # 5e: Distance to Exit
        side_dist = (exit_plane - float3(ipos) - local_pos) * inv_dir
        # Handle zero ray direction components
        side_dist = side_dist + (side_dist == 0.0) * 3e+38 # Note: source sets to 3e+38 if ray_dir component is 0, not side_dist. Correcting:
        side_dist = side_dist + (ray_dir == 0.0) * 3e+38

        # 5f: Find Minimum
        t = min(side_dist.x, min(side_dist.y, side_dist.z))
        crossed = step(side_dist, t) # This is a vector step function
        # Update crossed_axis based on weights of crossed axes
        # Note: source says "Update crossed_axis based on weights of the crossed axes."
        # This implies re-evaluating entry_weights or similar logic. For simplicity, we assume crossed_axis is updated to the axis of minimum t.
        crossed_axis = 0
        if crossed.y > 0.5:
            crossed_axis = 1
        if crossed.z > 0.5:
            crossed_axis = 2

        # 5g: Advance
        advanced = local_pos + max(t, 0.0001) * ray_dir
        cell_delta = int16_t3(floor(advanced + crossed * sign(ray_dir) * 0.5))
        ipos += cell_delta
        local_pos = advanced - float3(cell_delta)

def chunk_occupied(coord, chunk_masks, VX_MASK_EXT):
    local = coord & (VX_MASK_EXT - 1)
    bit = local.x + local.y * VX_MASK_EXT + local.z * VX_MASK_EXT * VX_MASK_EXT
    mask = chunk_masks.Load(int4(coord / VX_MASK_EXT, 0))
    return (mask & (1 << bit)) != 0

def brick_occupied(coord, brick_masks, VX_MASK_EXT):
    local = coord & (VX_MASK_EXT - 1)
    bit = local.x + local.y * VX_MASK_EXT + local.z * VX_MASK_EXT * VX_MASK_EXT
    mask = brick_masks.Load(int4(coord / VX_MASK_EXT, 0))
    return (mask & (1 << bit)) != 0

def voxel_occupied(coord, voxel_masks, VX_MASK_EXT):
    local = coord & (VX_MASK_EXT - 1)
    bit = local.x + local.y * VX_MASK_EXT + local.z * VX_MASK_EXT * VX_MASK_EXT
    mask = voxel_masks.Load(int4(coord / VX_MASK_EXT, 0))
    return (mask & (1 << bit)) != 0

def aadf_exit_plane(coord, cell_ext, aadf, ray_dir):
    shift_x = 0 if ray_dir.x < 0.0 else 5
    shift_y = 10 if ray_dir.y < 0.0 else 15
    shift_z = 20 if ray_dir.z < 0.0 else 25
    bounds_x = (aadf >> shift_x) & 31
    bounds_y = (aadf >> shift_y) & 31
    bounds_z = (aadf >> shift_z) & 31
    lower = max((coord - bounds_x) * cell_ext, 0)
    upper = min((coord + 1 + bounds_x) * cell_ext, grid_ext) # Note: grid_ext should be passed or global
    # This is a simplified version; source calculates lower/upper for each axis separately
    lower_x = max((coord.x - bounds_x) * cell_ext, 0)
    upper_x = min((coord.x + 1 + bounds_x) * cell_ext, grid_ext)
    lower_y = max((coord.y - bounds_y) * cell_ext, 0)
    upper_y = min((coord.y + 1 + bounds_y) * cell_ext, grid_ext)
    lower_z = max((coord.z - bounds_z) * cell_ext, 0)
    upper_z = min((coord.z + 1 + bounds_z) * cell_ext, grid_ext)
    exit_x = lower_x if ray_dir.x < 0.0 else upper_x
    exit_y = lower_y if ray_dir.y < 0.0 else upper_y
    exit_z = lower_z if ray_dir.z < 0.0 else upper_z
    return float3(exit_x, exit_y, exit_z)

def pack_voxel_cell(ipos, crossed_axis):
    # Packed cell info: voxel position and entry axis
    # Source says: "pack_voxel_cell(ipos, crossed_axis)"
    # Exact packing format not specified, but likely ipos in lower bits, axis in higher
    return (ipos.x | (ipos.y << 16) | (ipos.z << 32) | (crossed_axis << 60)) # Example packing
```

**Parameters:**
- `VX_MASK_EXT`: Side length of a mask block in voxels [S1, unofficial].
- `VX_BRICK_EXT`: Side length of a brick in voxels [S1, unofficial].
- `VX_CHUNK_EXT`: Side length of a chunk in voxels [S1, unofficial].
- `VX_NO_CELL`: Sentinel value returned when the ray misses the grid or exits the bounds [S1, unofficial].

**Edge Cases and Pitfalls:**
- Zero ray direction components cause division by zero in `inv_dir`. Handled by adding `1e-30` to the ray direction before inversion [S1, unofficial].
- Zero ray direction components cause infinite `side_dist`. Handled by setting the corresponding `side_dist` component to `3e+38` [S1, unofficial].
- Zero distance steps can cause infinite loops. Handled by using `max(t, 0.0001)` when advancing the ray [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Origin: Developed by Nelarius for the `vxray` project, an experimental voxel rendering engine using the DDA algorithm [S1, unofficial]. History: The method is part of the `vxray` codebase, which uses Metal and CMake for building and running [S1, unofficial]. Usage: Used in the `vxray` renderer for sparse voxel traversal [S1, unofficial].

## Sources
[S1] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
