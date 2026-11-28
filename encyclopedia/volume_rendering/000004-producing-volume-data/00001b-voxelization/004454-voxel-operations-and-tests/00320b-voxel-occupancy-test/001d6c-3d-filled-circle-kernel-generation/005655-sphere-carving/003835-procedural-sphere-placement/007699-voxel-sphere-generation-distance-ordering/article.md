# Voxel Sphere Generation & Distance Ordering

Also called: Bubble Generation, Voxel Sphere Carving

## Overview
This method generates a sparse, distance-ordered set of voxels representing a hollow or solid sphere by carving voids into a background field, building on [Procedural Sphere Placement](8171-procedural-sphere-placement.md). It differs from standard voxelization by explicitly filtering out occluded voxels via raycasting and organizing the remaining voxels into a sorted dictionary keyed by their distance from the center, enabling ordered rendering or growth animations.

## How it works
**Inputs:**
- `m_center`: World-space center of the sphere (`Vector3`).
- `m_maxRadius`: Maximum radius of the sphere in world units.
- `m_voxelScale`: Size of each voxel in world units.
- `VoxelResolution`: Resolution of the target grid (`Vector3Int`).
- `GlobalBounds`: Bounds of the scene/grid (`Bounds`).

**Outputs:**
- `VoxelsGrid`: A 1D `float[]` array of size $VoxelResolution.x \times VoxelResolution.y \times VoxelResolution.z$, storing `VoxelDistance` at computed grid indices.
- `m_orderedVoxels`: A `SortedDictionary<float, List<Voxel>>` mapping distance order to lists of voxels.
- `m_voxelsCache`: A `Dictionary<Vector3, Voxel>` for fast lookup of existing voxels.

**Data Structures:**
- `Voxel` struct:
  - `LocalPosition`: `Vector3` (local to sphere center).
  - `VoxelDistance`: `float` (Euclidean distance from center).
  - `Order`: `float` (used for sorting/animation timing).

**Parameters:**
- `m_growthTime`: Duration of the growth animation in seconds.
- `m_voxelScale`: Typical values depend on scene scale; must be consistent with `VoxelResolution` and `GlobalBounds`.

**Algorithm Steps:**

1.  **Calculate Integer Radius:**
    Compute the integer radius $r$ in voxels:
    $$ r = \text{round}\left(\frac{m\_maxRadius}{m\_voxelScale}\right) $$

2.  **Generate Octant Voxels:**
    Iterate $tx, ty, tz$ from $0$ to $r-1$.
    Compute squared magnitude:
    $$ \text{sqrMag} = \sqrt{tx^2 + ty^2 + tz^2} $$
    If $\text{sqrMag} < r$, invoke a callback with voxel at $(tx, ty, tz)$ and distance $\text{sqrMag}$.

3.  **Mirror to Full Sphere:**
    For each valid voxel in the first octant, generate symmetric counterparts to fill all 8 octants. The source specifies mirroring rules:
    - If $tx \neq 0$, add $(-tx, ty, tz)$.
    - If $tz \neq 0$, add $(-tx, ty, -tz)$.
    - If $tz \neq 0$, add $(tx, ty, -tz)$.
    - If $ty \neq 0$, add $(tx, -ty, tz)$.
    - Mirror remaining axes similarly to cover all sign combinations.
    *Note: The source logic implies generating all 8 octants by flipping signs of non-zero coordinates.*

4.  **Raycast Filtering:**
    For each generated voxel at local position `v.LocalPosition`:
    - Cast a `Physics.Raycast` from `m_center + Vector3.up * 0.1f` along `v.LocalPosition`.
    - Discard the voxel if the ray hits any geometry.
    - Add valid voxels to `m_voxelsCache` and `m_orderedVoxels`.
    - Key `m_orderedVoxels` by `Order = VoxelDistance`.
    - Track `maxRadius` and `outLayerVoxels` (voxels on the boundary).

5.  **Connectivity Expansion (Optional/Contextual):**
    While `deleted > 0`:
    - Iterate `outLayerVoxels`.
    - For each, get 6 neighbors via `GetConnectedVoxels`.
    - For each neighbor, check if not in cache and `!Physics.CheckBox(center + neighbor.pos + up*0.1f, scale*0.5)`.
    - If valid, set `VoxelDistance = parent.VoxelDistance + 1`, `Order = order`, add to cache/ordered dict, decrement `deleted`, increment `order`.
    - Update `maxRadius`. Break if `deleted == 0` or no connections found.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_voxel_sphere(m_center, m_max_radius, m_voxel_scale, voxel_resolution, global_bounds):
    # Step 1: Calculate integer radius
    r = round(m_max_radius / m_voxel_scale)
    
    voxels_cache = {}
    ordered_voxels = {}
    out_layer_voxels = []
    deleted_count = 0
    
    # Step 2 & 3: Generate and Mirror
    for tx in range(r):
        for ty in range(r):
            for tz in range(r):
                sqr_mag = sqrt(tx**2 + ty**2 + tz**2)
                
                if sqr_mag < r:
                    # Generate all 8 octants
                    positions = [
                        (tx, ty, tz),
                        (-tx, ty, tz) if tx != 0 else None,
                        (tx, -ty, tz) if ty != 0 else None,
                        (tx, ty, -tz) if tz != 0 else None,
                        (-tx, -ty, tz) if tx != 0 and ty != 0 else None,
                        (-tx, ty, -tz) if tx != 0 and tz != 0 else None,
                        (tx, -ty, -tz) if ty != 0 and tz != 0 else None,
                        (-tx, -ty, -tz) if tx != 0 and ty != 0 and tz != 0 else None
                    ]
                    
                    for pos in positions:
                        if pos is None:
                            continue
                        
                        local_pos = Vector3(pos[0] * m_voxel_scale, pos[1] * m_voxel_scale, pos[2] * m_voxel_scale)
                        
                        # Step 4: Raycast Filtering
                        ray_origin = m_center + Vector3(0, 0.1 * m_voxel_scale, 0)
                        ray_direction = local_pos.normalized
                        
                        if not physics_ray_cast(ray_origin, ray_direction):
                            voxel = Voxel(local_pos, sqr_mag, sqr_mag)
                            voxels_cache[local_pos] = voxel
                            
                            order_key = sqr_mag
                            if order_key not in ordered_voxels:
                                ordered_voxels[order_key] = []
                            ordered_voxels[order_key].append(voxel)
                            
                            out_layer_voxels.append(voxel)
                            deleted_count += 1 # Assuming initial state requires connection check
    
    # Step 5: Connectivity Expansion (if needed)
    while deleted_count > 0:
        current_layer_voxels = out_layer_voxels
        out_layer_voxels = []
        
        for voxel in current_layer_voxels:
            neighbors = get_6_neighbors(voxel.local_pos, m_voxel_scale)
            for neighbor_pos in neighbors:
                if neighbor_pos not in voxels_cache:
                    # Check bounding box
                    box_center = m_center + neighbor_pos + Vector3(0, 0.1 * m_voxel_scale, 0)
                    box_size = Vector3(m_voxel_scale, m_voxel_scale, m_voxel_scale)
                    
                    if not physics_check_box(box_center, box_size):
                        new_voxel = Voxel(neighbor_pos, voxel.voxel_distance + 1, voxel.order + 1)
                        voxels_cache[neighbor_pos] = new_voxel
                        
                        order_key = new_voxel.order
                        if order_key not in ordered_voxels:
                            ordered_voxels[order_key] = []
                        ordered_voxels[order_key].append(new_voxel)
                        
                        out_layer_voxels.append(new_voxel)
                        deleted_count -= 1
        
        if not out_layer_voxels:
            break
            
    # Populate VoxelsGrid
    voxels_grid = [0.0] * (voxel_resolution.x * voxel_resolution.y * voxel_resolution.z)
    for pos, voxel in voxels_cache.items():
        grid_idx = local_to_grid_index(pos, voxel_resolution, global_bounds)
        voxels_grid[grid_idx] = voxel.voxel_distance
        
    return voxels_grid, ordered_voxels, voxels_cache
```

## Operating characteristics
- **Time Complexity:** $O(r^3)$ for generation, where $r$ is the integer radius in voxels. Raycasting adds a cost proportional to the number of surface voxels.
- **Memory Complexity:** $O(N)$ where $N$ is the number of valid voxels after filtering. The `VoxelsGrid` is $O(VoxelResolution^3)$ but sparse in usage.
- **Strengths:** Explicit distance ordering allows for animated growth or ordered rendering. Raycasting ensures geometric accuracy against scene obstacles.
- **Limitations:** Raycasting is expensive for large spheres. The method is specific to Unity's physics system (`Physics.Raycast`, `Physics.CheckBox`), limiting portability.
- **When to use:** When dynamic, obstacle-aware voxel spheres are needed for effects like smoke or bubbles.
- **When not to use:** For static, high-resolution volumes where performance is critical and geometric accuracy against dynamic obstacles is not required.

## Background
This method originates from the "VolumetricSmoke" project by Voltonik, an unofficial implementation of voxel-based smoke rendering in Unity. It is part of a pipeline that includes procedural sphere placement and raymarching. The technique is designed for real-time visual effects in game engines.

## Sources
[S1] Voltonik/VolumetricSmoke, https://github.com/Voltonik/VolumetricSmoke, fetched 2026-10-28 (unofficial)
