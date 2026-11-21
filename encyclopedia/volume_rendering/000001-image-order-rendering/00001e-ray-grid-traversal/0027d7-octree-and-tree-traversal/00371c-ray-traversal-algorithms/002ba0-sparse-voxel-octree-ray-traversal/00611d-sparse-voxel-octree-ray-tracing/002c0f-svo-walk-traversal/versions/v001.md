# SVO Walk (Traversal)


## Overview
SVO Walk is a depth-first ray traversal algorithm for Sparse Voxel Octrees (SVO) that visits nodes in a specific order relative to the eye position, invoking a callback for each visited node [S1, unofficial]. It differs from pure SVO traversal by explicitly managing a stack to control visitation order based on eye position, and from BVH-based approaches by operating directly on the octree structure without an intermediate bounding volume hierarchy [S1, unofficial].

## How it works
**Inputs:** SVO root node, ray origin (typically $(0,0,0)$ in local space), ray direction, and a callback function `on_node`.
**Outputs:** Execution of `on_node` for each visited node in traversal order.

**Data Structures:**
*   **Stack:** A dynamic or fixed-size array storing tuples `(node_ref, stem, depth)`.
    *   `node_ref`: Pointer or index to the current octree node.
    *   `stem`: A 3D vector representing the integer coordinate of the node's origin at the current depth.
    *   `depth`: The current octree depth level (integer).
*   **Node:** Contains child pointers and data. Leaf nodes contain density/color data.

**Formulas:**
*   **Offset:** The physical size of a voxel at a given depth.
    $$ \text{offset} = 2^{-\text{depth}} $$
    Implemented via bit manipulation for efficiency:
    ```python
    exp = 126 - depth
    f = exp << 23
    offset = transmute_to_f32(f)
    ```
    [S1, unofficial]

*   **Voxel Position:** The world-space position of the voxel corner.
    $$ \text{voxel} = \text{stem} \times \text{offset} $$
    [S1, unofficial]

*   **Child Origin:** For a child at index $i$ (0-7) in the octree:
    $$ \text{new\_origin} = \text{stem} + \text{index\_offset}(i, \text{offset}) $$
    [S1, unofficial]

**Algorithm Steps:**
1.  Initialize the stack with `(ROOT_NODE, (0,0,0), 0)`. [S1, unofficial]
2.  While the stack is not empty:
    a.  Pop `(node_ref, stem, depth)` from the stack. [S1, unofficial]
    b.  Retrieve the node from `node_ref`. [S1, unofficial]
    c.  Calculate `offset = 2^(-depth)` using the bit-manipulation method above. [S1, unofficial]
    d.  Calculate `voxel = stem.extend(offset)` (i.e., `stem * offset`). [S1, unofficial]
    e.  Call `on_node(is_leaf, data, voxel)`. [S1, unofficial]
    f.  If `on_node` returns `true` AND the node is not a leaf:
        i.  Calculate the sorting order of the 8 children based on the eye position and `stem`. [S1, unofficial]
        ii. For each child in the sorted order:
            1.  Determine the child's index $i$.
            2.  Calculate `new_origin = stem + index_offset(i, offset)`. [S1, unofficial]
            3.  Push `(child_ref, new_origin, depth + 1)` to the stack. [S1, unofficial]

**Fast Walk Variant:**
*   Uses a fixed-size stack array of 256 elements. [S1, unofficial]
*   Uses raw pointers for node access to minimize overhead. [S1, unofficial]
*   Approximates voxel size calculation using `offset * 1.73` (approx $\sqrt{3}$) in some implementations, though the standard formula is $2^{-\text{depth}}$. [S1, unofficial]

**Edge Cases:**
*   **Stack Overflow:** The fixed-size stack (256 elements) may overflow in deep or complex scenes. [S1, unofficial]
*   **Empty Nodes:** The algorithm assumes `on_node` handles empty or invalid nodes appropriately. [S1, unofficial]
*   **Precision:** The bit-manipulation method for offset calculation is efficient but limited to specific depth ranges where $126 - \text{depth}$ yields a valid IEEE 754 exponent. [S1, unofficial]

## Subtopics

### [SVO Walk (Traversal) — fast_walk](5217-svo-walk-traversal-fast-walk.md)
A hybrid variant combining BVH coarse culling with SVO fine-grained evaluation.

## Operating characteristics
*   **Time Complexity:** Proportional to the number of visited nodes. Depth-first traversal with eye-position sorting ensures front-to-back rendering, allowing early termination if opacity thresholds are met. [S1, unofficial]
*   **Memory Complexity:** $O(D)$ for the stack, where $D$ is the maximum depth of the tree. The fixed-size stack variant uses constant memory (256 elements). [S1, unofficial]
*   **Strengths:** Simple implementation, low memory overhead, efficient for sparse data. [S1, unofficial]
*   **Limitations:** Fixed stack size may limit depth; no hierarchical culling beyond the octree structure itself (unlike BVH variants). [S1, unofficial]

## Background
The SVO Walk algorithm is described in the `svo-cpu` repository by marceline-cramer. It is an unofficial implementation of a depth-first SVO traversal technique. [S1, unofficial]

## Sources
[S1] marceline-cramer/svo-cpu, https://github.com/marceline-cramer/svo-cpu, fetched 2026-10-27 (unofficial)
