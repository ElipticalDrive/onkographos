# Hierarchical Tree Node Traversal

Also called: VolumeRayIntersector

## Overview
Hierarchical Tree Node Traversal is a method for accelerating ray-volume intersection by traversing a sparse octree structure to identify contiguous segments of active data. It solves the problem of inefficiently sampling empty space in large volumetric datasets by skipping inactive nodes and resolving hits at the node level rather than the individual voxel level [S1]. Unlike sibling methods that may compute precise Hermite data or rasterize shadows, this approach focuses on robust, thread-safe ray marching against a generic grid, providing entry and exit times for line-integration [S1].

## How it works
The method is implemented via the `VolumeRayIntersector` class template, which encapsulates the logic for intersecting a ray with a generic volume grid using an internal boolean tree [S1].

**Inputs and Outputs:**
*   **Input:** A `GridT` (the volume data), an optional `dilationCount` (integer, default 0), and an optional explicit bounding box `CoordBBox` [S1].
*   **Output:** Boolean flags indicating ray validity, and updated time parameters `t0` and `t1` representing the entry and exit points of active regions along the index ray [S1].

**Data Structures:**
*   **TreeT:** A boolean tree derived from the input grid's root type, converted to store `bool` values. This tree is used to accelerate ray marching and may be dilated based on `dilationCount` [S1].
*   **CoordBBox:** The axis-aligned bounding box of the active voxels in index space, cached for efficient ray-box intersection tests [S1].
*   **ValueAccessor:** An internal accessor for the tree, which is not thread-safe and must be copied per thread [S1].

**Parameters:**
*   `NodeLevel` (template int, default `GridT::TreeType::RootNodeType::ChildNodeType::LEVEL`): Specifies the tree node level at which traversal and hit resolution occur [S1].
*   `dilationCount` (int, default 0): The number of voxel dilations performed on a boolean copy of the input grid to account for interpolation kernel sizes [S1].

**Numbered Steps:**
1.  **Initialization:** Construct the intersector with the grid and optional dilation count or bounding box. The constructor validates that grid voxels are uniform and the grid is not empty, throwing a `RuntimeError` otherwise [S1].
2.  **Thread Safety Setup:** For multi-threaded execution, create a new instance for each computational thread using the shallow copy constructor. This ensures thread safety by duplicating the non-thread-safe `ValueAccessor` and cached `CoordBBox` efficiently [S1].
3.  **Ray Setup:** Call `setIndexRay` or `setWorldRay` with the ray to traverse. If the ray misses the grid's bounding box, the method returns `false`, indicating no further marching is required [S1].
4.  **Marching Loop:** Call `march(t0, t1)` in a loop. This function performs the hierarchical descent. It returns `true` if the ray intersects active values (active voxels or tiles) [S1].
5.  **Hit Resolution:** Upon a successful march, `t0` and `t1` are updated with the entry and exit times along the **INDEX** ray. These times are resolved at the node level (e.g., LeafNode) rather than for individual active voxels [S1].
6.  **Coordinate Conversion:** Convert the index-space hit times to world coordinates using `getIndexPos(time)`, `getWorldPos(time)`, and `getWorldTime(time)` [S1].
7.  **Hit List Generation:** Optionally, call `hits(list)` to populate a container of `RayType::TimeSpan` objects with all detected hits along the ray [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
# grid: The input volume grid
# dilation_count: Integer, default 0
intersector = VolumeRayIntersector(grid, dilation_count)

# Multi-threading: Copy for each thread
# thread_intersector = VolumeRayIntersector(intersector)

# Ray Setup
# ray: The ray object (index or world space)
if not intersector.setIndexRay(ray):
    # Ray misses the bounding box
    return

# Marching Loop
t0 = 0.0
t1 = 0.0

while intersector.march(t0, t1):
    # t0 and t1 are in index space
    # Perform line-integration or sampling between t0 and t1
    
    # Convert to world coordinates if needed
    world_pos_start = intersector.getWorldPos(t0)
    world_pos_end = intersector.getWorldPos(t1)
    
    # Prepare for next segment
    # Note: The implementation updates t0 and t1 internally for the next call
    # The loop continues until march() returns false

# Optional: Get all hits as a list
# hits_list = []
# intersector.hits(hits_list)
```

**Edge Cases and Pitfalls:**
*   **Thread Safety:** The class is not thread-safe due to its internal `ValueAccessor` and cached `CoordBBox`. Using a single instance across threads causes race conditions. Always use the copy constructor for each thread [S1].
*   **Coordinate Space:** `t0` and `t1` are computed in index space, not world space. Client code must use `getIndexPos` or maintain an index ray instance for coordinate conversion [S1].
*   **Uniform Voxels:** Constructors throw a `RuntimeError` if the grid voxels are not uniform or the grid is empty [S1].
*   **Node-Level Resolution:** Hits are resolved at the node level (e.g., LeafNode), not individual active voxels. This means `t0` and `t1` represent the bounds of a node, not necessarily the exact bounds of every active voxel within it [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the depth of the octree and the sparsity of the volume. Hierarchical traversal skips empty space, reducing the number of samples compared to fixed-step DDA.
*   **Memory Complexity:** Requires storage for the boolean tree (`TreeT`) and the bounding box cache. The shallow copy constructor ensures that per-thread memory overhead is limited to the accessor and bbox structures [S1].
*   **Strengths:** Thread-safe design via copying; handles generic grids; accounts for interpolation kernel sizes via dilation; robust bounding box checks [S1].
*   **Limitations:** Not thread-safe without copying; requires uniform voxels; hit resolution is at the node level, which may be coarser than voxel-level precision [S1].
*   **When to Use:** When rendering large, sparse volumetric datasets on multi-threaded CPU or GPU systems where thread safety and efficient empty space skipping are critical [S1].
*   **When Not to Use:** When voxel-level precision for hit times is required, or when the grid has non-uniform voxels [S1].

## Background
The `VolumeRayIntersector` class is part of the OpenVDB library, developed by DreamWorks Animation and later maintained by the Academy Software Foundation. It provides a standardized API for ray-volume intersection within the OpenVDB ecosystem, leveraging the library's sparse data structures for performance [S1].

## Sources
[S1] VolumeRayIntersector< GridT, NodeLevel, RayT > Class Template Reference, https://academysoftwarefoundation.github.io/openvdb/classopenvdb_1_1v9__0_1_1tools_1_1VolumeRayIntersector.html, fetched 2026-10-26
