# Spatial Hierarchies
Also called: Spatial Tree Structures
## Overview
Spatial hierarchies organize volumetric or geometric primitives into a tree structure to accelerate ray intersection tests and point location [S1, unofficial]. They solve the problem of $O(N)$ linear scans by enabling $O(\log N)$ traversal, reducing the number of samples required to render a volume [S1, unofficial]. Unlike flat grids, these structures adapt to data sparsity, skipping empty regions entirely [S1, unofficial].
## Shared principles
The family shares the mechanism of hierarchical descent and ascent. The traversal begins at the root of the octree and determines which of the eight children (octants) the ray intersects. This is typically done by comparing the ray's position relative to the center of the current node's bounding box [S1, unofficial][S2, unofficial][S6, unofficial][S8, unofficial].

1.  **Child Selection:** For a node with center $\mathbf{c}$ and half-extent $s$, the ray position $\mathbf{p}$ determines the child index $k \in \{0, \dots, 7\}$:
    $$
    k = 4 \cdot \mathbb{I}(p_x \ge c_x) + 2 \cdot \mathbb{I}(p_y \ge c_y) + 1 \cdot \mathbb{I}(p_z \ge c_z)
    $$
    where $\mathbb{I}(\cdot)$ is the indicator function [S1, unofficial][S2, unofficial][S6, unofficial][S8, unofficial].

2.  **Empty Space Skipping:** If the selected child is marked as empty (or uniform), the traversal skips the subtree. This is achieved by advancing the ray parameter $t$ to the exit of the current node or using neighbor pointers to jump to the next potentially occupied node [S1, unofficial][S6, unofficial][S8, unofficial][G].

3.  **Leaf Handling:** Upon reaching a leaf node, the method typically switches to a fixed-step traversal (like DDA) within the leaf's voxel grid to sample the volume data [S1, unofficial][S6, unofficial][S8, unofficial][G].

4.  **Backtracking:** Because octrees are sparse and irregular, traversal often requires backtracking to the parent node when a ray exits a subtree or encounters an empty branch. This is implemented using an explicit stack (on GPU) or recursion (on CPU) [S4, unofficial][S6, unofficial][S8, unofficial].
## Subtopics

### Octree
The most common spatial hierarchy for volumetric data, dividing space into eight octants recursively. It is chosen when data is sparse or anisotropic, offering better compression than uniform grids [S1, unofficial]. See [Octree and Tree Traversal](4134-octree-and-tree-traversal.md).

### BSP Tree
Binary Space Partitioning tree divides space using a single plane. It is chosen for static scenes with planar geometry or when exact partitioning is required, though it is less common for general volumetric data than octrees [S1, unofficial].

### KD Tree
k-Dimensional tree splits space along axes perpendicular to the coordinate axes. It is chosen for high-dimensional data or when axis-aligned splits are sufficient, often providing better cache locality than BSP trees [S1, unofficial].

### Insertion into Spatial Hierarchies
The process of adding primitives to the hierarchy. It uses a recursive insertion algorithm that subdivides nodes if they exceed a primitive count threshold [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def insert(node, prim):
    if overlap(node.bound, prim):
        if is_leaf(node):
            if node.nprims > MAXPRIMS and node.depth < MAXDEPTH:
                subdivide(node)
                for child in node.children:
                    insert(child, prim)
            else:
                list_insert(node.prims, prim)
        else:
            for child in node.children:
                insert(child, prim)
```

Parameters:
- **MAXDEPTH**: The maximum depth of the tree, typically 16 [S1, unofficial].
- **MAXPRIMS**: The maximum number of primitives allowed in a leaf node before subdivision, typically between 2 and 8 [S1, unofficial].

### Median-Cut
A bottom-up construction method that chooses the direction and position of splits carefully to balance the tree. It is chosen when a balanced tree is desired without iterative refinement [S1, unofficial].

### Surface Area Heuristic (SAH)
A cost model used to determine the optimal split position. It assumes the number of rays hitting an object is proportional to its projected surface area. It minimizes the expected cost of traversal by balancing the number of primitives against the surface area of the children [S1, unofficial].

## Background
Originated in computer graphics for ray tracing acceleration. The concept of spatial hierarchies like octrees, BSP trees, and KD trees was established in the early days of ray tracing to reduce intersection costs [S1, unofficial]. The insertion algorithm described is a standard top-down construction method [S1, unofficial].
## Sources
[S1] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
