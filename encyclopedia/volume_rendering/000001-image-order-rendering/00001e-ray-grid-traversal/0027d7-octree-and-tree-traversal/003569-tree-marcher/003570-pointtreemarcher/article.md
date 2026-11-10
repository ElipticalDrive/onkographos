# PointTreeMarcher

Also called: nanovdb::PointTreeMarcher

## Overview
`PointTreeMarcher` is a specialized tree traversal algorithm for point grids, implemented as a class template in OpenVDB's `nanovdb` library [S2]. It solves the problem of efficiently marching a ray through a hierarchical tree structure to find intersections with active nodes in a point-based volume representation [S1, S2]. It differs from the generic `TreeMarcher` by handling the specific coordinate system mismatch inherent in point grids, where data is bucketed into grid cells centered on voxels, while the underlying Digital Differential Analyzer (DDA) operates on grid nodes coincident with voxel corners [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** An accessor object (`AccT`) providing access to the grid data [S2]. A ray defined in index space (`RayT`) [S2].
*   **Output:** The `step` function returns a boolean indicating intersection. If true, it populates a pointer to the intersected `LeafNode` and the entry (`t0`) and exit (`t1`) times along the ray [S2].

**Data Structures:**
The class is a template with the following parameters [S2]:
*   `AccT`: The accessor type, providing access to the grid values [S1, S2].
*   `RayT`: The ray type, containing direction, origin, and inverse direction components [S1, S2].
*   `CoordT`: The coordinate type, defaulting to `Coord` [S1, S2].

Public type aliases defined within the class [S2]:
*   `ChildT`: `typename LeafNode< AccT::ValueType >::ChildNodeType`
*   `RealType`: `typename RayT::RealType`
*   `RealT`: `RealType`
*   `CoordType`: `CoordT`

**Algorithm Steps:**

1.  **Construction:** Initialize the `PointTreeMarcher` instance with a reference to the grid accessor [S2].
    ```python
    # Pseudocode for construction
    class PointTreeMarcher:
        def __init__(self, acc):
            self.acc = acc
    ```

2.  **Initialization:** Call `init` with the ray in index space [S2]. This step is critical for correctness [S2].
    *   The method applies an offset of $0.5$ to the ray origin (eye) [S2].
    *   **Reasoning:** In VDB point grids, points are bucketed into "grid cells" which are centered around grid voxels [S2]. Conversely, the underlying DDA (specifically the Half-Edge DDA or HDDA used in `nanovdb`) operates on "grid nodes" which are coincident with grid voxel corners [S2].
    *   Without this offset, the DDA would incorrectly operate on grid nodes instead of the intended grid cells [S2].
    *   The offset is applied to the ray origin rather than shifting the point data, ensuring the DDA intersects the correct grid cells [S2].
    ```python
    # Pseudocode for initialization
    def init(self, ray):
        # Apply 0.5 offset to ray origin to align DDA grid nodes with point grid cells
        # ray.origin is assumed to be a vector-like object
        ray.origin = ray.origin + 0.5
        self.ray = ray
        # Initialize internal DDA state based on the offset ray [S2]
        # Implementation details of DDA initialization are in the parent TreeMarcher/HDDA
    ```

3.  **Stepping:** Call `step` to advance the ray through the tree hierarchy [S2].
    *   The function uses the underlying HDDA to traverse the tree [S1].
    *   It checks for intersections with nodes of type `NodeT` (inherited from `TreeMarcher`, specifically `LeafNode` in this context) [S2].
    *   If an intersection is found, it updates the output parameters:
        *   `node`: Pointer to the intersected `LeafNode` [S2].
        *   `t0`: Entry time along the ray [S2].
        *   `t1`: Exit time along the ray [S2].
    *   Returns `true` if a node is intersected, `false` otherwise [S2].
    ```python
    # Pseudocode for stepping
    def step(self, node_ptr, t0, t1):
        # Advance ray using HDDA logic
        # Check if current ray segment intersects a valid node in the tree
        if intersection_found:
            node_ptr = current_node
            t0 = entry_time
            t1 = exit_time
            return True
        else:
            return False
    ```

**Parameters:**
*   `AccT`: Accessor type [S1, S2].
*   `RayT`: Ray type [S1, S2].
*   `CoordT`: Coordinate type, defaults to `Coord` [S1, S2].

**Edge Cases and Pitfalls:**
*   **Coordinate System Mismatch:** The primary pitfall is the difference between point bucketing (centered on grid voxels) and DDA grid nodes (coincident with grid voxels) [S2]. Failing to apply the $0.5$ offset in `init` results in the DDA operating on the wrong geometric primitives (nodes instead of cells), leading to incorrect traversal or missed intersections [S2].
*   **Precision:** The implementation uses `float` for `t0` and `t1` in the `step` signature, which may limit precision for very large or very small scenes compared to `double` [S2]. The ray type `RayT` defines its own `RealType` for internal calculations [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the tree depth and the ray path. The underlying HDDA provides efficient traversal, skipping empty space [S1].
*   **Memory Complexity:** Minimal overhead, primarily storing the ray state and accessor reference [S2].
*   **Strengths:** Correctly handles the specific coordinate semantics of point grids in VDB, which generic marchers might mishandle without manual offsetting [S2].
*   **Limitations:** Specific to `nanovdb` and point grids; not a general-purpose grid marcher [S1, S2].

## Background
`PointTreeMarcher` is part of the OpenVDB library, specifically within the `nanovdb` namespace, which is designed for high-performance, GPU-friendly volume rendering [S1, S2]. It builds on the generic `TreeMarcher` and the Half-Edge DDA (HDDA) implementation [S1, S2]. The class was introduced to address the nuances of marching through point-based representations where data is stored in cells rather than nodes [S2].

## Sources
[S1] nanovdb/util/HDDA.h File Reference - OpenVDB, https://academysoftwarefoundation.github.io/openvdb/HDDA_8h.html, fetched 2026-10-25
[S2] PointTreeMarcher< AccT, RayT, CoordT > Class Template Reference, http://academysoftwarefoundation.github.io/openvdb/classnanovdb_1_1PointTreeMarcher.html, fetched 2026-10-26
