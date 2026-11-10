# K-D Tree Hierarchy Construction

Also called: Polygon distribution in k-d tree, Spatial partitioning with k-d trees

## Overview
K-D Tree Hierarchy Construction is a method for building a spatial partitioning structure to accelerate volume rendering by grouping polygons (or volumetric primitives) into a binary tree. It solves the problem of efficiently determining which primitives contribute to a pixel without explicit clipping, by storing polygon IDs in a prefix-order traversal array [S1]. It sets itself apart from its parent method, [Orthogonal BSP Tree Construction](1134-orthogonal-bsp-tree-construction.md), by using a standard k-d tree structure where splitting planes are orthogonal to coordinate axes, enabling trivial visibility sorting for 3D texture rendering [S1]. Unlike fixed viewpoint partitioning, this method is view-independent, making it suitable for animation sequences where the viewpoint changes [S1].

## How it works
**Inputs:** A set of polygons (or volumetric primitives) defined by their vertices in 3D space. A user-defined threshold $N_{leaf}$ for the maximum number of polygons per leaf node.

**Outputs:** A k-d tree structure where each node $v$ contains a splitting dimension $i$, a splitting value $X_i$, and a list of polygon IDs. The polygons are stored in a common array `Tid` arranged in prefix-order traversal order [S1].

**Data Structures:**
*   **Node:** Contains `split_dim` (integer, axis index 0, 1, or 2), `split_val` (float, position along axis), `left_child` (index or pointer), `right_child` (index or pointer), and `poly_ids` (list of integers).
*   **Tid Array:** A flat array storing polygon IDs. The position of a polygon in `Tid` corresponds to its visitation order during a prefix-order traversal of the tree. This allows all polygons associated with a subtree to be passed in one operation, avoiding linked lists [S1].

**Algorithm Steps:**
1.  **Initialize:** Start with the root node containing all input polygons.
2.  **Check Leaf Condition:** If the number of polygons in the current node $v$ is less than or equal to $N_{leaf}$, mark $v$ as a leaf node and store the polygon IDs in `Tid` according to the current traversal index. Return.
3.  **Select Splitting Dimension:** Choose a dimension $i$ to split on. The implementation described uses a round-robin fashion (alternating dimensions $x, y, z$) [S1].
4.  **Determine Splitting Value:** Choose a value $X_i$ for the splitting hyperplane. The implementation bisects the region (median or midpoint) [S1]. The hyperplane is orthogonal to the $x_i$-axis.
5.  **Partition Polygons:** For each polygon $p$ in the current node:
    *   Let $V_p$ be the set of vertices of $p$.
    *   If $\forall v \in V_p, v.x_i < X_i$, place $p$ in the set for the left subtree.
    *   If $\forall v \in V_p, v.x_i > X_i$, place $p$ in the set for the right subtree.
    *   Otherwise, retain $p$ in the set stored at node $v$ (the node itself) [S1].
    *   *Note:* Polygons are associated with the smallest node region that completely contains them, avoiding subdivision of polygons [S1].
6.  **Recurse:** Recursively construct the left and right subtrees using the respective polygon sets.
7.  **Store IDs:** As nodes are visited in prefix order, append the polygon IDs stored at each node to the `Tid` array [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_kd_tree(polygons, depth=0, threshold=10):
    """
    Recursively builds a k-d tree.
    polygons: list of polygon objects, each having a list of vertices.
    depth: current tree depth, used for round-robin axis selection.
    threshold: minimum number of polygons to split.
    Returns: a Node object.
    """
    node = Node()
    node.poly_ids = []
    
    # Base case: leaf node
    if len(polygons) <= threshold:
        # Store polygon IDs in the node
        node.poly_ids = [p.id for p in polygons]
        return node

    # Step 1: Select splitting dimension (round-robin)
    split_dim = depth % 3
    
    # Step 2: Determine splitting value (bisect region)
    # Find min and max extent along split_dim
    min_val = min(v.coords[split_dim] for p in polygons for v in p.vertices)
    max_val = max(v.coords[split_dim] for p in polygons for v in p.vertices)
    split_val = (min_val + max_val) / 2.0
    
    node.split_dim = split_dim
    node.split_val = split_val
    
    # Step 3: Partition polygons
    left_polygons = []
    right_polygons = []
    node_polygons = []
    
    for p in polygons:
        all_less = True
        all_greater = True
        
        for v in p.vertices:
            coord = v.coords[split_dim]
            if coord >= split_val:
                all_less = False
            if coord <= split_val:
                all_greater = False
                
        if all_less:
            left_polygons.append(p)
        elif all_greater:
            right_polygons.append(p)
        else:
            # Polygon straddles the plane or lies on it; keep at node
            node_polygons.append(p)
            
    # Step 4: Recurse
    node.left_child = build_kd_tree(left_polygons, depth + 1, threshold)
    node.right_child = build_kd_tree(right_polygons, depth + 1, threshold)
    
    return node

def flatten_tree(node, tid_array, index_ref):
    """
    Flattens the k-d tree into a prefix-order traversal array of IDs.
    """
    if node is None:
        return
        
    # Append current node's polygons
    for pid in node.poly_ids:
        tid_array[index_ref[0]] = pid
        index_ref[0] += 1
        
    # Recurse left then right
    flatten_tree(node.left_child, tid_array, index_ref)
    flatten_tree(node.right_child, tid_array, index_ref)
```

**Parameters:**
*   $N_{leaf}$: User-defined threshold for leaf node creation. Typical values are not specified in the sources, but it controls the granularity of the partitioning [S1].
*   Splitting Policy: Round-robin axis selection is used in the cited implementation [S1]. More sophisticated policies based on object locations are possible but not detailed [S1].

**Edge Cases and Pitfalls:**
*   **Polygon Association:** Polygons are associated with the smallest node region that completely contains them. This avoids subdividing polygons but means some polygons may be stored at internal nodes if they straddle multiple split planes [S1].
*   **Precision:** The sources do not specify floating-point precision issues, but bisecting regions requires careful handling of vertices lying exactly on the split plane (handled by retaining them at the node) [S1].

## Subtopics

### [Real-time KD Tree Construction](4437-real-time-kd-tree-construction.md)
Real-time KD Tree Construction is a technique for building k-d tree spatial partitioning structures on the GPU to enable immediate use in rendering pipelines. It sets itself apart from its parent method, [K-D Tree Hierarchy Construction](2300-k-d-tree-hierarchy-construction.md), by utilizing GPU parallelism and breadth-first search (BFS) to achieve real-time performance. It differs from [GPU K-D Tree Ray Traversal](4575-gpu-k-d-tree-ray-traversal.md) by focusing on the *construction* phase rather than the *traversal* phase [G].

### [GPU K-D Tree Ray Traversal](4575-gpu-k-d-tree-ray-traversal.md)
GPU K-D Tree Ray Traversal accelerates ray tracing on graphics hardware by intersecting rays with a hierarchical spatial partitioning structure (K-D tree). It efficiently skips empty space and identifies intersected primitives, avoiding the computational cost of testing every primitive in the scene. It is distinguished from CPU implementations by its adaptation to GPU architectures: early Shader Model 2.0 hardware used stackless multi-pass streaming kernels, while Shader Model 3.0 introduced single-pass traversal via dynamic branching. It builds on [K-D Tree Hierarchy Construction](2300-k-d-tree-hierarchy-construction.md).

### [Parallel K-d Tree Construction](5421-parallel-k-d-tree-construction.md)
Parallel K-d Tree Construction is a cooperative parallel partitioning process that uses a k-d tree to group data cells into local regions, restoring spatial ordering to allow early ray merging and efficient rendering in distributed-memory systems. It sets itself apart from static partitioning methods by dynamically determining split planes via global consensus and managing neighbor links to handle boundary conditions, ensuring that processors render cells in the same neighborhood at approximately the same time.

### [K-D Tree Subdivision](8148-k-d-tree-subdivision.md)
K-D Tree Subdivision partitions a volumetric dataset into subvolumes using a k-D tree structure with alternating binary subdivision of coordinate planes, enabling unambiguous front-to-back ordering for compositing. Unlike fixed viewpoint partitioning which minimizes overlap by slicing orthogonal to the view, k-D tree partitioning is view-independent and suitable for animation sequences where the viewpoint changes.

### Axis Selection via Variance
**Definition:** A variant method for selecting the splitting axis for a K-D tree node by analyzing the variance of the QEF (Quadratic Error Function) solution along each dimension, scaled by the box size [S2, unofficial].

**Steps:**
1.  Solve QEF to get an approximate solution and error [S2, unofficial].
2.  Calculate variance for each axis: $variance = qef.getVariance(approximate)$. Scale variance by box size: $variance[i] *= size[i]$ [S2, unofficial].
3.  Find $maxVarDir$ (index of max variance) and $minVarDir$ (index of min variance) [S2, unofficial].
4.  Set $dir = maxVarDir$. If $size[maxVarDir] < 2$, fallback to $3 - maxVarDir - minVarDir$. If that size $< 2$, fallback to $minVarDir$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. Standard k-d tree construction is $O(N \log N)$ for balanced trees, but the sources do not specify the complexity for this specific implementation [S1].
*   **Memory Complexity:** The use of a common array `Tid` avoids the extra space of linked lists, storing polygon IDs in prefix-order traversal order [S1].
*   **Performance:** The implementation suggests scalability up to about 16 processors before contention becomes a serious drawback [S1].
*   **Strengths:** Avoids explicit clipping of polygons; polygons are associated with the smallest node region that completely contains them, avoiding subdivision [S1].
*   **Limitations:** The round-robin splitting policy may not be optimal for all scenes; more sophisticated policies based on object locations are possible but not implemented in the cited work [S1].

## Background
The k-d tree was introduced by Jon Bentley [S1]. The specific implementation described in the sources is from a 1996 paper on hierarchical and parallelizable direct volume rendering for irregular and multiple grids [S1]. The method builds on earlier work by Greene [S1] regarding polygon association with node regions.

## Sources
[S1] Hierarchical and parallelizable direct volume rendering for irregular and multiple grids, https://doi.org/10.1109/visual.1996.567606, fetched 2026-10-24
[S2] Danielmelody/KdtreeISO, https://github.com/Danielmelody/KdtreeISO, fetched 2026-10-27 (unofficial)
