# Tree Marcher

Also called: Tree Marcher for Generic Grids

## Overview
The Tree Marcher is a hierarchical ray traversal algorithm designed to step a ray through a generic grid structure (such as OpenVDB) to identify intersections with active nodes [S1, S2]. It solves the problem of efficiently navigating sparse, multi-resolution volumetric data by skipping empty subtrees and returning precise intersection intervals for occupied nodes [S1]. Unlike sibling methods that may rely on specific rasterization techniques or fixed-step DDA for the entire volume, the Tree Marcher operates at the node level of the hierarchy, serving as the bridge between coarse hierarchical culling and fine-grained voxel sampling [S1, S2]. It is distinct from [PointTreeMarcher](7535-pointtreemarcher.md), which handles coordinate system mismatches specific to point-based grids [S2].

## How it works
The Tree Marcher relies on the parent's [Shared principles](4134-octree-and-tree-traversal.md) for hierarchical descent, empty space skipping, and backtracking. It utilizes a Hierarchical Digital Differential Analyzer (HDDA) internally to manage the ray's progression through the tree structure [S2].

### Inputs and Outputs
*   **Inputs**:
    *   `acc`: An accessor object providing access to the grid data [S1].
    *   `indexRay`: A ray defined in index (grid) space, containing origin and direction [S1].
*   **Outputs**:
    *   `node`: A pointer to the intersected node of type `NodeT` [S1].
    *   `t0`: The start parameter of the intersection interval within the node [S1].
    *   `t1`: The end parameter of the intersection interval within the node [S1].

### Data Structures
The `TreeMarcher` is a class template with the following signature [S1]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
template<typename NodeT, typename RayT, typename AccT, typename CoordT = Coord>
class TreeMarcher;
```
*   **Template Parameters**:
    *   `NodeT`: The type of node being traversed [S2].
    *   `RayT`: The ray type, containing ray data and a `RealType` for precision [S1].
    *   `AccT`: The accessor type used for grid access [S1].
    *   `CoordT`: The coordinate type, defaulting to `Coord` [S1].
*   **Internal Typedefs**:
    *   `ChildT`: Defined as `typename NodeT::ChildNodeType` [S1].
    *   `RealType`: Defined as `typename RayT::RealType` [S1].
    *   `RealT`: Alias for `RealType` [S1].
    *   `CoordType`: Alias for `CoordT` [S1].

### Algorithm Steps
The traversal is controlled by two primary methods: `init` and `step` [S1].

1.  **Initialization**:
    *   Construct the `TreeMarcher` with the grid accessor `acc` [S1].
    *   Call `init(indexRay)` to initialize the internal state with the index-space ray [S1]. This prepares the HDDA for traversal [S2].

2.  **Stepping**:
    *   Call `step(node, t0, t1)` repeatedly [S1].
    *   The method advances the ray through the tree hierarchy.
    *   If the ray intersects a node of type `NodeT`:
        *   Populate `node` with a pointer to the intersected node [S1].
        *   Populate `t0` with the start of the intersection interval [S1].
        *   Populate `t1` with the end of the intersection interval [S1].
        *   Return `true` [S1].
    *   If the ray exits the grid or encounters no further nodes of type `NodeT`:
        *   Return `false` [S1].

### Pseudocode
The following pseudocode demonstrates the usage pattern. It assumes the existence of the `TreeMarcher` class and its methods as defined in the sources [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Assume AccT, NodeT, RayT, CoordT are defined types
# acc is an instance of AccT
# ray is an instance of RayT in index space

# 1. Construction
tree_marcher = TreeMarcher(acc)

# 2. Initialization
tree_marcher.init(ray)

# 3. Traversal Loop
while True:
    node_ptr = None
    t0 = 0.0
    t1 = 0.0
    
    # Attempt to step to the next node
    hit = tree_marcher.step(node_ptr, t0, t1)
    
    if not hit:
        # No more nodes intersected by the ray
        break
    
    # Process the intersected node
    # node_ptr points to the intersected NodeT
    # t0 and t1 define the interval [t0, t1] along the ray
    process_node(node_ptr, t0, t1)
```

### Parameters and Edge Cases
*   **Precision**: The ray type `RayT` defines the `RealType` used for calculations [S1]. Precision issues may arise if the ray origin or direction is not properly normalized or scaled to index space before initialization [G].
*   **Empty Space**: The traversal relies on the underlying grid's metadata to skip empty nodes. If the grid structure is corrupted or metadata is inconsistent, the traversal may fail or return incorrect intervals [G].
*   **Node Type**: The `NodeT` template parameter restricts the traversal to specific node types (e.g., internal nodes vs. leaf nodes). If the ray passes through nodes of a different type, they are skipped [S1].

## Subtopics

### [PointTreeMarcher](7535-pointtreemarcher.md)
A specialized variant for point grids that resolves coordinate system mismatches between bucketed grid cells and voxel corners. Choose this when working with point-based volumes in OpenVDB's `nanovdb` library.

## Operating characteristics
*   **Time Complexity**: The traversal time is proportional to the number of nodes intersected by the ray. Empty space skipping reduces the number of visited nodes significantly in sparse volumes [S1, S2].
*   **Memory Complexity**: The algorithm uses minimal stack space, relying on the internal state of the `TreeMarcher` object. It does not require an explicit stack for backtracking in the same way recursive methods do, as the HDDA manages the traversal state [S2].
*   **Strengths**: Efficient for sparse, multi-resolution data. Generic interface allows reuse across different node types and grid structures [S1].
*   **Limitations**: Requires the grid to be accessible via an accessor. Performance depends on the quality of the tree structure (e.g., balance, depth) [G].
*   **When to Use**: Use when rendering sparse volumetric data stored in OpenVDB or similar hierarchical grids where node-level intersection information is required.
*   **When Not to Use**: Avoid for dense, uniform grids where a simple DDA is more efficient. Avoid if point-grid specific coordinate handling is needed (use [PointTreeMarcher](7535-pointtreemarcher.md) instead) [S2].

## Background
The Tree Marcher is part of the OpenVDB library, specifically within the `nanovdb` namespace, which provides a compact, GPU-friendly representation of VDB grids [S1, S2]. It was developed to support efficient volume rendering and simulation on modern hardware by leveraging hierarchical data structures [S1]. The class is documented in the OpenVDB API reference [S1].

## Sources
[S1] OpenVDB: TreeMarcher< NodeT, RayT, AccT, CoordT > Class Template Reference, https://academysoftwarefoundation.github.io/openvdb/classnanovdb_1_1TreeMarcher.html, fetched 2026-10-24
[S2] nanovdb/util/HDDA.h File Reference - OpenVDB, https://academysoftwarefoundation.github.io/openvdb/HDDA_8h.html, fetched 2026-10-25
