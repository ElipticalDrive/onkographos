# Hierarchical Tree Node Traversal

Also called: VolumeRayIntersector

## Overview
Hierarchical Tree Node Traversal is a ray-intersection method that accelerates volume rendering by traversing a boolean tree structure representing the active regions of a volume grid. It solves the problem of efficiently skipping empty space and coarse structures during ray marching. Unlike sibling methods that may traverse uniform grids or specific brick layouts, this approach operates on a generic hierarchical tree (typically an octree or similar sparse structure) where nodes represent tiles or voxels, allowing for variable-resolution traversal. It is implemented in OpenVDB as the `VolumeRayIntersector` class.

## How it works
**Inputs and Outputs:**
- **Input:** A generic volume grid (`GridT`) containing scalar or vector values, and a ray (`RayT`) defined in either index space or world space.
- **Output:** A sequence of time intervals (`t0`, `t1`) along the ray where active data exists, and the corresponding world positions.

**Data Structures:**
- **TreeT:** A boolean tree derived from the input grid's tree structure. It is defined as `tree::Tree<typename RootType::template ValueConverter<bool>::Type>`. This tree stores only the occupancy (active/inactive) of nodes, not the actual values. It may be dilated by `dilationCount` voxels to account for interpolation kernel support [S1].
- **CoordBBox:** The axis-aligned bounding box of the active voxels in index space, cached for efficient ray-box intersection tests [S1].
- **ValueAccessor:** An internal accessor for the boolean tree, used during traversal. It is not thread-safe [S1].

**Parameters:**
- **dilationCount (int, default 0):** The number of voxel dilations applied to the boolean tree. This ensures that rays passing near active voxels (which might be sampled due to interpolation) are not missed [S1].
- **NodeLevel (template int, default `GridT::TreeType::RootNodeType::ChildNodeType::LEVEL`):** Specifies the tree node level at which hit resolution occurs. Hits are resolved at this level (e.g., LeafNode) rather than at individual voxels [S1].

**Steps:**
1. **Construction:** Create an instance of the intersector with the grid and optional dilation count or bounding box. If the grid voxels are not uniform or the grid is empty, a `RuntimeError` is thrown [S1].
2. **Thread Safety:** For multi-threaded execution, create a new instance for each thread using the copy constructor. This is required because the internal `ValueAccessor` and cached `CoordBBox` are not thread-safe, but copying is efficient [S1].
3. **Ray Setup:** Call `setIndexRay` or `setWorldRay` with the ray to traverse. If the ray misses the grid's bounding box, the method returns `false`, indicating no further marching is required [S1].
4. **Marching Loop:** Call `march(t0, t1)` in a loop. This function returns `true` if the ray intersects active values (active voxels or tiles). It updates `t0` and `t1` with the entry and exit times along the **index ray** [S1].
   - `t0`: Time of the first hit of an active tile or leaf.
   - `t1`: Time of the first hit (> t0) of an inactive tile or exit point of the BBOX for leaf nodes [S1].
   - Note: `t0` and `t1` are computed in index space, not world space [S1].
5. **Hit Retrieval:** Use `hits(list)` to populate a container of `RayType::TimeSpan` objects if detailed hit information is needed [S1].
6. **Coordinate Conversion:** Convert index-space hit times to world coordinates using `getIndexPos(time)`, `getWorldPos(time)`, and `getWorldTime(time)` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume GridT, RayT, and RealType are defined
# Assume grid is a valid, uniform, non-empty volume grid

# 1. Construction (Master Thread)
inter = VolumeRayIntersector(grid, dilationCount=0)

# 2. Thread Safety: Copy for each worker thread
# inter_thread = VolumeRayIntersector(inter) # Shallow copy

# 3. Ray Setup
ray = RayT(origin, direction) # Defined in world or index space
if not inter.setWorldRay(ray):
    return # Ray misses grid

# 4. Marching Loop
t0 = 0.0
t1 = 0.0
while inter.march(t0, t1):
    # t0 and t1 are in index space
    # Perform line-integration or sampling between t0 and t1
    
    # Convert to world coordinates if needed
    pos_start = inter.getWorldPos(t0)
    pos_end = inter.getWorldPos(t1)
    time_start = inter.getWorldTime(t0)
    time_end = inter.getWorldTime(t1)
    
    # Prepare for next iteration
    # Note: t0 and t1 are updated by march() for the next segment
    pass

# 5. Hit Retrieval (Optional)
hit_list = []
inter.hits(hit_list)
```

**Edge Cases and Pitfalls:**
- **Thread Safety:** The class is not thread-safe due to its internal `ValueAccessor` and cached `CoordBBox`. Copying is efficient and required per thread [S1].
- **Coordinate Space:** `t0` and `t1` are computed in index space. Client code must use `getIndexPos` or maintain an index ray instance for coordinate conversion [S1].
- **Uniform Voxels:** Constructors throw `RuntimeError` if the grid voxels are not uniform or the grid is empty [S1].
- **Hit Resolution:** Hits are resolved at the node level (e.g., LeafNode) rather than individual active voxels. This means `t0` and `t1` may encompass multiple voxels within a leaf node [S1].

## Operating characteristics
- **Time Complexity:** Depends on the tree depth and the number of active nodes intersected by the ray. Traversal is logarithmic with respect to the grid size due to the hierarchical structure.
- **Memory Complexity:** Requires storage for the boolean tree (`TreeT`), which is typically smaller than the full value grid. The `CoordBBox` is cached [S1].
- **Strengths:** Efficiently skips empty space and coarse structures. Thread-safe when using copy constructor. Handles generic volume grids.
- **Limitations:** Not thread-safe without copying. Requires uniform voxels. Hit resolution is at node level, not voxel level.
- **When to Use:** When rendering sparse volumes with significant empty space, or when using OpenVDB grids.
- **When Not to Use:** When voxel-level precision is required for hit times, or when the grid is dense and uniform (DDA might be simpler).

## Background
The method is implemented in OpenVDB as the `VolumeRayIntersector` class template. It was designed to provide a public API for intersecting rays with generic volumes, internally performing hierarchical tree node traversal. The class is part of the OpenVDB tools library [S1].

## Sources
[S1] VolumeRayIntersector< GridT, NodeLevel, RayT > Class Template Reference, https://academysoftwarefoundation.github.io/openvdb/classopenvdb_1_1v9__0_1_1tools_1_1VolumeRayIntersector.html, fetched 2026-10-26
