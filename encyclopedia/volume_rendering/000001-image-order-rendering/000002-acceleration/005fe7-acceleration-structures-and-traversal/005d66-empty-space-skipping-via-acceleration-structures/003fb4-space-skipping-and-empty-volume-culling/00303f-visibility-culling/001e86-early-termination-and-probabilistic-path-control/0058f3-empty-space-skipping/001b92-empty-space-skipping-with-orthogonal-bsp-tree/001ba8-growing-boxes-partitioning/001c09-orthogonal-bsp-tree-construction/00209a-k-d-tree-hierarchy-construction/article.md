# K-D Tree Hierarchy Construction

Also called: Polygon distribution in k-d tree, Spatial partitioning with k-d trees

## Overview
K-D Tree Hierarchy Construction builds a binary space-partitioning structure to organize geometric primitives (typically polygons) for efficient spatial queries, such as visibility culling in volume rendering [S1]. It sets itself apart from its parent method, [Orthogonal BSP Tree Construction](1134-orthogonal-bsp-tree-construction.md), by classifying polygons based on vertex coordinates relative to the splitting hyperplane and retaining spanning polygons at the node, thereby avoiding explicit polygon clipping [S1].

## How it works
**Inputs:** A set of polygons $P$, each defined by vertices in 3D space [S1]. A user-defined leaf threshold $T_{leaf}$ [S1].
**Outputs:** A binary tree where each node $v$ contains a list of polygon IDs and a splitting dimension $i$ (if internal) [S1].

**Data Structures:**
*   **Tree Node:** Contains `split_dim` (integer, 0-2), `split_val` (float), `left_child` (pointer/index), `right_child` (pointer/index), and `poly_ids` (list of integers) [S1].
*   **Polygon ID Array (`Tid`):** A flat array storing polygon IDs in prefix-order traversal sequence, allowing all polygons in a subtree to be passed in one operation without linked lists [S1].

**Algorithm:**
1.  **Root Initialization:** Create the root node [G]. Assign all input polygons to the root's `poly_ids` list [G].
2.  **Recursive Build Function `Build(node, polygons, depth)`:**
    a.  **Leaf Check:** If `len(polygons) <= T_leaf`, mark `node` as a leaf [S1]. Return [G].
    b.  **Splitting Dimension Selection:** Determine dimension $i$ to split on [S1]. The basic implementation uses round-robin selection (e.g., $i = depth \mod 3$) [S1]. More sophisticated policies may use variance analysis [S2, unofficial].
    c.  **Split Value Selection:** Determine the splitting value $X_i$ [S1]. The basic implementation bisects the region along dimension $i$ [S1].
    d.  **Polygon Classification:** Initialize `left_polys` and `right_polys` as empty lists [G]. For each polygon $p$ in `polygons`:
        i.  Check all vertices of $p$ [S1].
        ii. If all vertices have coordinate $x_i < X_i$, add $p$ to `left_polys` [S1].
        iii. If all vertices have coordinate $x_i > X_i$, add $p$ to `right_polys` [S1].
        iv. Otherwise, retain $p$ in `node.poly_ids` [S1]. This associates the polygon with the smallest node region that completely contains it, avoiding subdivision [S1].
    e.  **Recursive Step:**
        i.  If `left_polys` is not empty, create `node.left_child` and call `Build(node.left_child, left_polys, depth + 1)` [G].
        ii. If `right_polys` is not empty, create `node.right_child` and call `Build(node.right_child, right_polys, depth + 1)` [G].
        iii. If both are empty, the node is a leaf (or contains only overlapping polygons) [S1].
3.  **Serialization:** Traverse the tree in prefix order to populate the `Tid` array [S1].

**Parameters:**
*   $T_{leaf}$: Typical values are not given in the sources [G]. Common practice suggests 1-10 depending on cache size [G].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point comparisons for $x_i < X_i$ and $x_i > X_i$ must be consistent to avoid polygons being lost or duplicated [G].
*   **Empty Subtrees:** If a split results in no polygons on one side, that child should not be created to save space [G].
*   **Degenerate Input:** Polygons with zero area or vertices exactly on the split plane are handled by the "retain in node" rule (case iv), ensuring they are not lost [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Node:
    def __init__(self):
        self.split_dim = -1
        self.split_val = 0.0
        self.left_child = None
        self.right_child = None
        self.poly_ids = []
        self.is_leaf = False

def build_kd_tree(polygons, depth=0, leaf_threshold=4):
    """
    Builds a K-D tree hierarchy.
    polygons: List of dicts with 'id' and 'vertices' (list of tuples).
    depth: Current tree depth.
    leaf_threshold: Minimum number of polygons to trigger a split [S1].
    Returns: Node object.
    """
    node = Node()
    node.poly_ids = [p['id'] for p in polygons]

    # Leaf check [S1]
    if len(polygons) <= leaf_threshold:
        node.is_leaf = True
        return node

    # Splitting dimension selection (Round-robin as per [S1])
    split_dim = depth % 3

    # Split value selection (Bisection as per [S1])
    min_val = float('inf')
    max_val = float('-inf')
    for p in polygons:
        for v in p['vertices']:
            val = v[split_dim]
            if val < min_val: min_val = val
            if val > max_val: max_val = val

    split_val = (min_val + max_val) / 2.0
    node.split_dim = split_dim
    node.split_val = split_val
    node.poly_ids = [] # Reset to store IDs of polygons retained at this node

    left_polys = []
    right_polys = []

    for p in polygons:
        all_less = True
        all_greater = True
        for v in p['vertices']:
            val = v[split_dim]
            if val >= split_val:
                all_less = False
            if val <= split_val:
                all_greater = False

        if all_less:
            left_polys.append(p)
        elif all_greater:
            right_polys.append(p)
        else:
            node.poly_ids.append(p['id']) # Retain spanning polygons [S1]

    # Recursive step [G]
    if left_polys:
        node.left_child = build_kd_tree(left_polys, depth + 1, leaf_threshold)
    if right_polys:
        node.right_child = build_kd_tree(right_polys, depth + 1, leaf_threshold)

    node.is_leaf = (node.left_child is None and node.right_child is None)
    return node

def serialize_tree(node, tid_array):
    """Traverses the tree in prefix order to populate the Tid array [S1]."""
    if node is None:
        return
    tid_array.extend(node.poly_ids)
    serialize_tree(node.left_child, tid_array)
    serialize_tree(node.right_child, tid_array)
```

## Subtopics

### [Real-time KD Tree Construction](4437-real-time-kd-tree-construction.md)
Real-time KD Tree Construction is a technique for building k-d tree spatial partitioning structures on the GPU to enable immediate use in rendering pipelines. It sets itself apart from its parent method, [K-D Tree Hierarchy Construction](2300-k-d-tree-hierarchy-construction.md), by utilizing GPU parallelism and breadth-first search (BFS) to achieve real-time performance. It differs from [GPU K-D Tree Ray Traversal](4575-gpu-k-d-tree-ray-traversal.md) by focusing on the *construction* phase rather than the *traversal* phase [G].

### [GPU K-D Tree Ray Traversal](4575-gpu-k-d-tree-ray-traversal.md)
GPU K-D Tree Ray Traversal accelerates ray tracing on graphics hardware by intersecting rays with a hierarchical spatial partitioning structure (K-D tree). It efficiently skips empty space and identifies intersected primitives, avoiding the computational cost of testing every primitive in the scene. It is distinguished from CPU implementations by its adaptation to GPU architectures: early Shader Model 2.0 hardware used stackless multi-pass streaming kernels, while Shader Model 3.0 introduced single-pass traversal via dynamic branching. It builds on [K-D Tree Hierarchy Construction](2300-k-d-tree-hierarchy-construction.md) [G].

### [Parallel K-d Tree Construction](5421-parallel-k-d-tree-construction.md)
Parallel k-d tree construction algorithms partition sparse volumetric or unstructured data into spatial regions to enable efficient ray traversal, early termination, and workload distribution in parallel volume rendering. Unlike static partitioning, these methods dynamically determine split planes using occupancy queries or median selection to balance tree depth and node count. They differ from [Real-time KD Tree Construction](4437-real-time-kd-tree-construction.md) by focusing on distributed-memory or GPU-accelerated top-down recursion rather than single-node heuristic optimization, and from [K-D Tree Subdivision](8148-k-d-tree-subdivision.md) by explicitly managing inter-processor communication or GPU kernel synchronization during the build phase.

### [K-D Tree Subdivision](8148-k-d-tree-subdivision.md)
K-D Tree Subdivision partitions a 3D volume into subvolumes using a k-d tree structure with alternating binary subdivision of coordinate planes, enabling unambiguous front-to-back ordering for compositing. Unlike fixed viewpoint partitioning which minimizes overlap by slicing orthogonal to the view, k-D tree partitioning is view-independent and suitable for animation sequences where the viewpoint changes. It determines the front-to-back image compositing order hierarchically by recursive traversal, visiting the 'front' child before the 'back' child.

### Axis Selection via Variance
Selects the splitting axis for a K-D tree node by analyzing the variance of the QEF (Quadratic Error Function) solution along each dimension, scaled by the box size [S2, unofficial]. The process involves solving the QEF to get an approximate solution and error, then calculating variance for each axis: $variance[i] = qef.getVariance(approximate) \times size[i]$ [S2, unofficial]. The axis with maximum variance is chosen (`dir = maxVarDir`). Fallback logic applies if the box size along the selected axis is less than 2, choosing `3 - maxVarDir - minVarDir`, and if that is also small, falling back to `minVarDir` [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [G]. Average case is $O(N \log N)$ for balanced trees, where $N$ is the number of polygons; worst case is $O(N^2)$ for degenerate inputs [G].
*   **Memory Complexity:** $O(N)$ for storing polygons and tree nodes; the flat `Tid` array avoids overhead of linked lists [S1].
*   **Strengths:** Avoids explicit polygon clipping, reducing computational cost during rendering [S1]. Efficient for static scenes [G].
*   **Limitations:** Construction can be slow for very large datasets [G]. Sensitive to the choice of split plane and leaf threshold [G].
*   **When to use:** Static scenes with many polygons where visibility culling is critical [G].
*   **When not to use:** Highly dynamic scenes where frequent tree rebuilds are prohibitive [G].

## Background
The k-d tree was introduced by Jon Bentley [S1]. It was originally designed for efficient spatial partitioning in computer graphics, specifically for hidden surface removal and ray tracing [G]. The method described here, which stores polygons in nodes rather than splitting them, is a variation that avoids the need for explicit clipping, as noted in early volume rendering implementations [S1].

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
[S2] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
