# Linear Octree with Z-order curve (Morton tree)
Also called: Morton tree, Linear Octree, Z-order Octree

## Overview
A storage method for sparse octrees that stores nodes in a linear array generated using a Z-order curve, eliminating the storage of inner nodes [S1, unofficial]. It is designed for raycasting and voxel applications where the tree can be precalculated and resources are limited, such as GPU textures [S1, unofficial]. Unlike the parent [Morton-coded Sparse Voxel Tree](5212-morton-coded-sparse-voxel-tree.md) which retains a hierarchical mask structure, this method flattens the tree to store only relevant data, trading traversal speed for memory efficiency [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** A sparse octree structure containing leaf nodes with associated data (e.g., density, color).
**Outputs:** A linear array of leaf nodes sorted by their Morton code, and optionally a secondary data texture for voxel attributes.

**Data Structures:**
1.  **Linear Array:** A contiguous memory block storing leaf nodes. Each entry contains the spatial coordinates (or Morton code) and the voxel data.
2.  **Data Texture (Optional):** A secondary texture or array holding the actual voxel values, referenced by the linear array indices [S1, unofficial].

**Steps:**
1.  **Traverse and Collect:** Traverse the sparse octree and collect all leaf nodes. Inner nodes are discarded [S1, unofficial].
2.  **Compute Morton Code:** For each leaf node at depth $d$ with coordinates $(x, y, z)$, compute its Morton code (Z-order curve index). The Morton code interleaves the bits of the coordinates. For a node at depth $d$ in a tree of max depth $D$, the code is typically scaled to fit the coordinate space [S2, unofficial].
3.  **Sort:** Sort the collected leaf nodes based on their Morton codes in ascending order [S2, unofficial].
4.  **Store:** Write the sorted leaf nodes into the linear array [S2, unofficial].
5.  **Access:** To find a voxel, perform a binary search on the linear array using the target Morton code [S2, unofficial]. If the voxel data is stored separately, use the found index to access the data texture [S1, unofficial].

**Formulas:**
*   **Morton Code ($M$):** For a coordinate $(x, y, z)$, the Morton code is formed by interleaving the bits of $x, y, z$.
    $$ M(x, y, z) = \text{interleave}(x, y, z) $$
*   **Lookup Complexity:** $O(\log_2(L))$, where $L$ is the number of leaf nodes [S2, unofficial].

**Parameters:**
*   **Tree Depth:** Determined by the resolution of the volume data.
*   **Leaf Count ($L$):** The number of non-empty leaf nodes in the sparse octree.

**Edge Cases and Pitfalls:**
*   **Inner Node Elimination:** Because inner nodes are eliminated from the primary storage, accessing voxel data requires cross-referencing the linear tree array with a second data texture [S1, unofficial].
*   **Neighbor Finding:** The structure does not store interior nodes or pointers, making it unsuitable for applications requiring traversal of non-leaf structures or neighbor finding without additional data [S2, unofficial].
*   **Search Speed:** The binary search is generally slower than pointer-based octree traversal ($O(\log_8(N))$) except in highly vertical trees with few leaf nodes relative to interior nodes [S2, unofficial].

## Subtopics

### Linear Octree Hashing
A pointerless octree structure where only leaf nodes are stored, allocated contiguously in memory, and sorted by locational code to enable binary search [S2, unofficial]. It performs point location by executing a binary search on the sorted array of leaves [S2, unofficial]. Leaf nodes are sorted by Morton interleaved base-8 locational code and laid out sequentially in memory [S2, unofficial]. It does not store interior nodes or pointers, making it unsuitable for applications requiring traversal of non-leaf structures or neighbor finding without additional data [S2, unofficial]. Node lookup complexity is $O(\log_2(L))$ where $L$ is the number of leaf nodes [S2, unofficial]. It is slower to hash than pointer octrees ($O(\log_2(L))$ vs $O(\log_8(N))$) except in highly vertical trees with few leaf nodes relative to interior nodes [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Node lookup is $O(\log_2(L))$ where $L$ is the number of leaf nodes [S2, unofficial]. This is generally slower than pointer octrees ($O(\log_8(N))$) except in highly vertical trees with few leaf nodes relative to interior nodes [S2, unofficial].
*   **Memory Complexity:** Requires storage of neither pointers nor interior nodes, making it highly memory efficient [S2, unofficial].
*   **Strengths:** Ideal for applications where storage is of utmost importance and only leaf nodes are needed [S2, unofficial]. Suitable for GPU textures where memory bandwidth and capacity are limited [S1, unofficial].
*   **Limitations:** Slower lookup speed compared to pointer-based structures. Not suitable for dynamic updates or neighbor finding without additional data structures [S2, unofficial].
*   **When to Use:** When memory is constrained, the tree can be precalculated, and only leaf node access is required (e.g., static volume rendering) [S1, unofficial][S2, unofficial].
*   **When Not to Use:** When frequent neighbor lookups or dynamic tree modifications are required [S2, unofficial].

## Background
The linear octree was originally proposed by Gargantini for quadtrees [S2, unofficial]. It was later extended to octrees using Morton codes [S2, unofficial]. While not directly applied to volume rendering in the surveyed literature, it is considered an intriguing structure for its potential compression abilities in adaptive octree methods on volume data [S2, unofficial].

## Sources
[S1] Efficient storage for a sparse octree?, https://stackoverflow.com/questions/6085209/efficient-storage-for-a-sparse-octree, fetched 2026-10-26 (unofficial)
[S2] A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
