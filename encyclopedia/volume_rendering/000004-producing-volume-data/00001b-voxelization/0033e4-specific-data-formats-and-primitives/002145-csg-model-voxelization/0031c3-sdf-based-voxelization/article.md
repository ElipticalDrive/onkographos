# SDF-based Voxelization

Also called: Phase 1 voxelization, Voxelization of CSG Meshes and Analytic Primitives

## Overview
SDF-based Voxelization is a method to visualize any Constructive Solid Geometry (CSG) mesh or analytic primitive as a Minecraft-style voxel chunk pile by determining world-space bounding boxes, sampling a 3D grid, and querying Signed Distance Fields (SDF) to determine occupancy [S1, unofficial]. It differs from GPU-based rasterization voxelization by relying on CPU-side SDF queries and storing results in a packed bitmap for later rendering as analytic box primitives, enabling A/B comparison with the original mesh without shader changes [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A CSG mesh or analytic primitive object, identified by an ID.
*   **Output:** A `VoxelGrid` data structure containing a packed bitmap of occupied voxels, and optionally an on-disk binary cache file.

**Data Structures**
The core data structure is `VoxelGrid`. Its layout includes:
*   `object_id`: Identifier for the source object.
*   `voxel_size`: The side length of the voxels in meters.
*   `AABB`: The world-space Axis-Aligned Bounding Box of the object.
*   `bitmap`: A packed bitmap representing the occupancy of voxels within the AABB. The size of the bitmap is determined by the dimensions of the AABB divided by `voxel_size`, rounded up.

**Parameters**
*   `r_voxel_size`: The voxel side length in meters. Default value is 0.1 [S1, unofficial].
*   `r_voxelize_demo`: A toggle flag. When 1, the system voxelize the current scene's CSG meshes and renders them as voxels instead of the original mesh for A/B comparison. Default is 0 [S1, unofficial].

**Algorithm Steps**
1.  **Bounding Box Calculation:** For each object (analytic primitive or CSG mesh), determine its world-space Axis-Aligned Bounding Box (AABB) [S1, unofficial].
2.  **Grid Initialization:** Define a 3D grid covering the AABB with a spacing equal to `r_voxel_size` [S1, unofficial].
3.  **SDF Query:** For each voxel center in the grid:
    *   Query the Signed Distance Field (SDF) of the object at that center point.
    *   For analytic primitives, use existing renderer SDF primitive types.
    *   For non-SDF sources (e.g., triangle meshes), build a quick triangle-mesh-to-SDF helper [S1, unofficial].
4.  **Occupancy Test:** A voxel is considered "inside" the object if its SDF value is less than half the voxel size:
    $$ SDF < \frac{voxel\_size}{2} $$
    If true, set the corresponding bit in the `VoxelGrid` bitmap to 1; otherwise, 0 [S1, unofficial].
5.  **Caching:** Store the resulting `VoxelGrid` in an on-disk ad-hoc binary file for caching. The cache key is composed of the object hash and the `voxel_size` [S1, unofficial].
6.  **Rendering:** For rendering, each occupied voxel becomes an analytic box primitive in the renderer's primitive list. The primitive inherits the source object's material at voxelization time. The voxelized objects are rendered via the existing path tracer without requiring shader changes [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def voxelize_object(object_id, voxel_size):
    # 1. Get object and its AABB
    obj = get_object(object_id)
    aabb = compute_world_aabb(obj)
    
    # 2. Determine grid dimensions
    dims = aabb.size / voxel_size
    grid_dims = ceil(dims)
    
    # 3. Initialize VoxelGrid
    voxel_grid = VoxelGrid(
        object_id=object_id,
        voxel_size=voxel_size,
        aabb=aabb,
        bitmap_size=grid_dims.x * grid_dims.y * grid_dims.z
    )
    
    # 4. Sample SDF and populate bitmap
    for x in range(grid_dims.x):
        for y in range(grid_dims.y):
            for z in range(grid_dims.z):
                # Calculate voxel center in world space
                center = aabb.min + vec3(x + 0.5, y + 0.5, z + 0.5) * voxel_size
                
                # Query SDF
                # For analytic primitives, use built-in SDF function
                # For meshes, use mesh-to-SDF helper
                dist = query_sdf(obj, center)
                
                # Check occupancy
                if dist < voxel_size / 2:
                    set_bit(voxel_grid.bitmap, x, y, z, 1)
                else:
                    set_bit(voxel_grid.bitmap, x, y, z, 0)
    
    # 5. Cache result
    cache_key = compute_cache_key(object_id, voxel_size)
    save_to_disk(voxel_grid, cache_key)
    
    return voxel_grid

def render_voxel_grid(voxel_grid):
    # 6. Create analytic box primitives for occupied voxels
    primitives = []
    for x in range(voxel_grid.aabb.size.x / voxel_grid.voxel_size):
        for y in range(voxel_grid.aabb.size.y / voxel_grid.voxel_size):
            for z in range(voxel_grid.aabb.size.z / voxel_grid.voxel_size):
                if get_bit(voxel_grid.bitmap, x, y, z):
                    center = voxel_grid.aabb.min + vec3(x + 0.5, y + 0.5, z + 0.5) * voxel_grid.voxel_size
                    box = AnalyticBox(center, vec3(voxel_grid.voxel_size))
                    box.material = get_material(voxel_grid.object_id)
                    primitives.append(box)
    
    # 7. Add to renderer primitive list
    add_primitives_to_renderer(primitives)
```

**Edge Cases and Pitfalls**
*   **Single Voxel Size:** The implementation supports only a single voxel size per object. Adaptive resolution is excluded from this phase [S1, unofficial].
*   **Static Position:** Voxelization is a CPU job. Source object positions are read once at voxelization time. If the source moves, the voxel grid does not follow until re-voxelized [S1, unofficial].
*   **Precision:** The occupancy test uses a threshold of half the voxel size. This may lead to aliasing or gaps if the SDF query is not precise enough, though SDF queries are generally considered cheap and robust for this purpose [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of voxels in the AABB. Each voxel requires one SDF query.
*   **Memory Complexity:** $O(N)$ for the packed bitmap.
*   **Performance:** Voxelization is a one-time precompute on the CPU. SDF queries are cheap. Cache hits speed up repeated voxelization [S1, unofficial].
*   **Scaling:** A 1m³ cube voxelized at 0.1m resolution produces approximately 1000 voxel boxes [S1, unofficial].
*   **Strengths:** Simple implementation, no shader changes required, enables A/B comparison with original mesh, caching improves performance for repeated use.
*   **Limitations:** Single voxel size per object, static voxel grids (no animation support in this phase), CPU-bound (no GPU voxelization), no chunk merging or optimization [S1, unofficial].
*   **When to use:** When visualizing CSG meshes or analytic primitives as voxel chunks for destruction or aesthetic purposes, and when the objects are static or voxelization can be recomputed.
*   **When not to use:** When real-time voxelization of moving objects is required, or when adaptive resolution is needed [S1, unofficial].

## Background
SDF-based Voxelization is part of the "voxel destruction roadmap" for the demont-engine, specifically Phase 1. It was introduced in GitHub Issue #140 by havokentity [S1, unofficial]. The method lays the data structures that subsequent phases (simulation, chunk merging, fracture patterns, VFX) build upon. It is implemented in `src/destruction/Voxelizer.cpp/h` and `src/destruction/VoxelGrid.cpp/h` [S1, unofficial].

## Sources
[S1] destruction: voxelize CSG mesh + render as voxels (Phase 1 of voxel destruction roadmap), https://github.com/havokentity/demont-engine/issues/140, fetched 2026-10-26 (unofficial)
