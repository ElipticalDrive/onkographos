# Hierarchical Tree Node Traversal

Also called: VolumeRayIntersector

## Overview
Hierarchical Tree Node Traversal accelerates ray-volume intersection by traversing a sparse octree to identify contiguous segments of active data [S1]. It solves the problem of inefficiently sampling empty space by skipping inactive nodes and resolving hits at the node level rather than the individual voxel level [S1]. Unlike sibling methods that rasterize shadows or compute precise Hermite data, this approach focuses on robust ray marching against a generic grid, providing entry and exit times for line-integration [S1].

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
1.  **Initialization:** Construct the intersector with the grid and optional dilation count or bounding box [S1]. The constructor validates that grid voxels are uniform and the grid is not empty, throwing a `RuntimeError` otherwise [S1].
2.  **Thread Safety Setup:** For multi-threaded execution, create a new instance for each computational thread using the shallow copy constructor [S1]. This ensures thread safety by duplicating the non-thread-safe `ValueAccessor` and cached `CoordBBox` efficiently [S1].
3.  **Ray Setup:** Call `setIndexRay` or `setWorldRay` with the ray to traverse [S1]. If the ray misses the grid's bounding box, the method returns `false`, indicating no further marching is required [S1].
4.  **Marching Loop:** Call `march(t0, t1)` in a loop [S1]. This function performs the hierarchical descent [S1]. It returns `true` if the ray intersects active values (active voxels or tiles) [S1].
5.  **Hit Resolution:** Upon a successful march, `t0` and `t1` are updated with the entry and exit times along the **INDEX** ray [S1]. These times are resolved at the node level (e.g., LeafNode) rather than for individual active voxels [S1].
6.  **Coordinate Conversion:** Convert the index-space hit times to world coordinates using `getIndexPos(time)`, `getWorldPos(time)`, and `getWorldTime(time)` [S1].
7.  **Hit List Generation:** Optionally, call `hits(list)` to populate a container of `RayType::TimeSpan` objects with all detected hits along the ray [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Ray:
    def __init__(self, origin, direction):
        self.origin = origin
        self.direction = direction

class CoordBBox:
    def __init__(self, min_coord, max_coord):
        self.min = min_coord
        self.max = max_coord

def ray_intersects_bbox(ray, bbox):
    t_min = (bbox.min[0] - ray.origin[0]) / ray.direction[0]
    t_max = (bbox.max[0] - ray.origin[0]) / ray.direction[0]
    if t_min > t_max: t_min, t_max = t_max, t_min
    t_y = (bbox.min[1] - ray.origin[1]) / ray.direction[1]
    if t_y > t_max: return False
    if t_y < t_min: t_min = t_y
    t_z = (bbox.min[2] - ray.origin[2]) / ray.direction[2]
    if t_z > t_max: return False
    if t_z < t_min: t_min = t_z
    return t_min <= t_max

class VolumeRayIntersector:
    def __init__(self, grid, dilation_count=0, bbox=None):
        # Validates uniform voxels; throws RuntimeError if not uniform or empty
        self.grid = grid
        self.dilation_count = dilation_count
        self.bbox = bbox if bbox else self._compute_active_bbox(grid)
        self.tree = self._build_bool_tree(grid, dilation_count)
        self.accessor = self._create_accessor(self.tree)
        self.index_ray = None

    def _compute_active_bbox(self, grid):
        return CoordBBox(grid.min_index, grid.max_index)

    def _build_bool_tree(self, grid, dilation_count):
        # Creates boolean tree from grid, dilated by dilation_count
        # Not given in the sources.
        return []

    def _create_accessor(self, tree):
        return {}

    def set_index_ray(self, ray):
        self.index_ray = ray
        return ray_intersects_bbox(ray, self.bbox)

    def march(self, t0, t1):
        # Performs hierarchical descent through the boolean tree
        # Updates t0 and t1 with entry/exit times along the INDEX ray
        # Returns True if ray intersects active values (tiles or leaves)
        # Not given in the sources.
        return False

    def get_index_pos(self, time):
        return [self.index_ray.origin[i] + time * self.index_ray.direction[i] for i in range(3)]

    def get_world_pos(self, time):
        # Converts index-space time to world position using grid transform
        # Not given in the sources.
        return self.get_index_pos(time)

    def get_world_time(self, time):
        # Converts index-space time to world-space time
        # Not given in the sources.
        return time

    def hits(self, list_out):
        # Populates list with RayType::TimeSpan objects
        # Not given in the sources.
        pass

# Usage loop
intersector = VolumeRayIntersector(grid, dilation_count=0)
thread_intersector = VolumeRayIntersector(intersector)
ray = Ray(origin=[0.0, 0.0, 0.0], direction=[1.0, 0.0, 0.0])
if not thread_intersector.set_index_ray(ray):
    pass
t0 = 0.0
t1 = 0.0
while thread_intersector.march(t0, t1):
    world_pos_start = thread_intersector.get_world_pos(t0)
    world_pos_end = thread_intersector.get_world_pos(t1)
    # Perform line-integration between t0 and t1
hits_list = []
thread_intersector.hits(hits_list)
```

**Edge Cases and Pitfalls:**
*   **Thread Safety:** The class is not thread-safe due to its internal `ValueAccessor` and cached `CoordBBox` [S1]. Using a single instance across threads causes race conditions [S1]. Always use the copy constructor for each thread [S1].
*   **Coordinate Space:** `t0` and `t1` are computed in index space, not world space [S1]. Client code must use `getIndexPos` or maintain an index ray instance for coordinate conversion [S1].
*   **Uniform Voxels:** Constructors throw a `RuntimeError` if the grid voxels are not uniform or the grid is empty [S1].
*   **Node-Level Resolution:** Hits are resolved at the node level (e.g., LeafNode), not individual active voxels [S1]. This means `t0` and `t1` represent the bounds of a node, not necessarily the exact bounds of every active voxel within it [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the depth of the octree and the sparsity of the volume [G]. Hierarchical traversal skips empty space, reducing the number of samples compared to fixed-step DDA [G].
*   **Memory Complexity:** Requires storage for the boolean tree (`TreeT`) and the bounding box cache [S1]. The shallow copy constructor ensures that per-thread memory overhead is limited to the accessor and bbox structures [S1].
*   **Strengths:** Thread-safe design via copying; handles generic grids; accounts for interpolation kernel sizes via dilation; robust bounding box checks [S1].
*   **Limitations:** Not thread-safe without copying; requires uniform voxels; hit resolution is at the node level, which may be coarser than voxel-level precision [S1].
*   **When to Use:** When rendering large, sparse volumetric datasets on multi-threaded CPU or GPU systems where thread safety and efficient empty space skipping are critical [S1].
*   **When Not to Use:** When the grid has non-uniform voxels, as the constructor throws a `RuntimeError` [S1].

## Background
The `VolumeRayIntersector` class is part of the OpenVDB library, developed by DreamWorks Animation and later maintained by the Academy Software Foundation [S1]. It provides a standardized API for ray-volume intersection within the OpenVDB ecosystem, leveraging the library's sparse data structures for performance [S1].

## Sources
[S1] VolumeRayIntersector< GridT, NodeLevel, RayT > Class Template Reference, https://academysoftwarefoundation.github.io/openvdb/classopenvdb_1_1v9__0_1_1tools_1_1VolumeRayIntersector.html, fetched 2026-10-26
