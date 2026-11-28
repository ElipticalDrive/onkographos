# Leaf Collection


## Overview
Leaf Collection identifies and gathers all terminal nodes (leaves) intersected by a ray grid during hierarchical traversal, converting a sparse traversal path into a contiguous list of active volumetric blocks for subsequent ray marching or shading. It separates node identification from radiance computation to enable efficient memory compaction and parallel processing.

## How it works
**Inputs:**
- A hierarchical sparse voxel structure (e.g., Octree) with nodes classified as Empty, Full, Branch, or Leaf [G].
- A set of rays (ray grid) defined by origin and direction [G].
- Traversal state tracking the current node and depth [G].

**Outputs:**
- A list (vector) of leaf node identifiers (coordinates, indices, or pointers) intersected by the rays [G].

**Data Structures:**
- **Node Classification:** Each node has a type field: `EMPTY`, `FULL`, `BRANCH`, `LEAF` [G].
- **Result Vector:** A dynamic array storing tuples of `(leaf_id, depth)` or `(x, y, z, depth)` [G].

**Numbered Steps:**
1. **Initialize** an empty result vector $L$ [G].
2. **For each ray** $r$ in the ray grid:
    a. Start traversal at the root node $N$ [G].
    b. **While** $N$ is not a terminal node (i.e., $N$ is `BRANCH`):
        i. Determine which child of $N$ is intersected by $r$ [G].
        ii. Let $C$ be the intersected child.
        iii. Update traversal state (e.g., current coordinates, depth) [G].
        iv. Set $N \leftarrow C$ [G].
    c. **If** $N$ is `LEAF`, `FULL`, or `EMPTY`:
        i. Push $N$'s coordinates and depth to $L$ [S1, unofficial].

**Formulas:**
- Child coordinate update: $x_c = 2x_p + b_x$, $y_c = 2y_p + b_y$, $z_c = 2z_p + b_z$, where $(x_p, y_p, z_p)$ are the parent coordinates, and $(b_x, b_y, b_z)$ are the octant bits derived from the child index [S1, unofficial].

**Parameters:**
Not given in the sources.

**Edge Cases:**
Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def collect_leaves(node, current_coords, depth, result_list):
    """
    Recursively collect leaf nodes from a hierarchical structure.
    Implements the logic from [S1, unofficial].
    """
    node_type = node.type
    if node_type in ('EMPTY', 'FULL', 'LEAF'):
        result_list.append((current_coords, depth))
    elif node_type == 'BRANCH':
        for octant in range(8):
            child = node.children[octant]
            child_coords = (
                current_coords[0] * 2 + (octant & 1),
                current_coords[1] * 2 + ((octant >> 1) & 1),
                current_coords[2] * 2 + ((octant >> 2) & 1)
            )
            collect_leaves(child, child_coords, depth + 1, result_list)
```

## Subtopics

### [Leaf Node Representation](3000-leaf-node-representation.md)
The Leaf Node Representation defines the data structure and logic for terminal nodes in sparse voxel octree traversal. It encapsulates volumetric sample data alongside validity and leaf flags, providing methods for validity checking and color manipulation during ray marching.

### [Leaf Offset Computation](5185-leaf-offset-computation.md)
Leaf Offset Computation calculates the linear memory offsets for each leaf node in an OpenVDB tree to facilitate parallel processing and data copying. It serves as a prerequisite for [Leaf Processing](5189-leaf-processing.md), which uses these offsets to copy precomputed voxel data from a contiguous source buffer into leaf node buffers. Unlike [Leaf Node Representation](3000-leaf-node-representation.md), which defines the internal structure of a leaf, this method computes the external indexing required to map leaf nodes to a flat memory space.

### Rendered Feedback Information Collection
This variant integrates leaf collection with level-of-detail (LOD) management and feedback mechanisms for streaming [S2, unofficial].
- **Step:** During tree traversal, stop when reaching the node corresponding to the needed level-of-detail [S2, unofficial]. If required data is present, traverse the mipmapped brick and collect the current node index in a node-list [S2, unofficial]. If the appropriate level-of-detail is missing or the node is terminal (no more refined data in the original model), add it to the node-list [S2, unofficial]. If the node needs further subdivision, add it to the node-list and indicate subdivision need via one bit in the node [S2, unofficial]. Push rendered nodes into a FIFO during traversal [S2, unofficial]. Shift the 48-element window over a small set of frames to increase retrievable nodes [S2, unofficial].
- **Parameter:** In the first frame, stop pushing after 48 elements [S2, unofficial]. In the second frame, stop pushing after 96 elements [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of nodes visited during traversal [G].
- **Memory Complexity:** $O(L)$ where $L$ is the number of collected leaves [G].
- **Strengths:** Decouples traversal from shading, allowing for efficient batch processing and memory compaction [G]. Enables adaptive LOD and streaming by identifying exactly which nodes need to be loaded or updated [G].
- **Limitations:** Requires an additional pass or buffer to store the collected leaves [G]. If the volume is dense, the list can become large, potentially causing memory pressure [G].
- **When to Use:** Ideal for sparse voxel octrees where most of the space is empty, and efficient memory access patterns are critical [G]. Also suitable for streaming scenarios where only relevant nodes need to be transferred to the GPU or CPU [G].
- **When Not to Use:** For dense volumes where traversal overhead is negligible compared to shading, or when hardware supports direct hierarchical traversal without intermediate collection [G].

## Background
Not given in the sources.

## Sources
[S1] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S2] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
