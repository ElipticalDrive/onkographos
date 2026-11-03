# Leaf Collection


## Overview
Leaf Collection extracts terminal nodes from a hierarchical spatial structure into a linear list for batch processing [G]. It decouples traversal from ray-casting, differing from methods like [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) that integrate traversal into rasterization [G].

## How it works
**Inputs:** A root node of a hierarchical structure (e.g., octree) [G].
**Outputs:** A vector/list of terminal node coordinates and their associated depth levels [G].

**Data Structures:**
*   **Result Vector:** A dynamic array storing tuples of `(x, y, z, depth)` [G].
*   **Node:** Each node contains a type indicator (Empty, Full, Branch, Leaf) and, if a Branch, pointers or indices to 8 children [G].

**Steps:**
1.  Initialize an empty result vector [G].
2.  Call the recursive collection function on the root node with initial coordinates `(0, 0, 0)` and depth `0` [G].
3.  **Recursive Function `Collect(node, cx, cy, cz, depth)`:**
    1.  Determine the type of `node` [G].
    2.  If `node` is **Empty**, **Full**, or **Leaf**:
        *   Append `(cx, cy, cz, depth)` to the result vector [G].
    3.  If `node` is a **Branch**:
        *   Iterate through its 8 children (octants 0 to 7) [G].
        *   For each child `i` (where `i` is the octant index):
            *   Calculate child coordinates:
                $$cx_{child} = cx \times 2 + (i \& 1)$$
                $$cy_{child} = cy \times 2 + ((i >> 1) \& 1)$$
                $$cz_{child} = cz \times 2 + ((i >> 2) \& 1)$$
            *   Recursively call `Collect(child, cx_{child}, cy_{child}, cz_{child}, depth + 1)` [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def collect_leaves(node, cx, cy, cz, depth, result):
    """Recursively collects terminal nodes from a hierarchical tree."""
    # Traversal logic inferred from standard octree conventions [G].
    if node.type in ('EMPTY', 'FULL', 'LEAF'):
        result.append((cx, cy, cz, depth))
    elif node.type == 'BRANCH':
        for i in range(8):
            bit_x = i & 1
            bit_y = (i >> 1) & 1
            bit_z = (i >> 2) & 1
            new_cx = cx * 2 + bit_x
            new_cy = cy * 2 + bit_y
            new_cz = cz * 2 + bit_z
            collect_leaves(node.children[i], new_cx, new_cy, new_cz, depth + 1, result)
```

**Parameters:**
*   **Depth Limit:** Not given in the sources. [G] Typically capped by memory or scene bounds.
*   **Coordinate System:** Integer grid coordinates. [G]

**Edge Cases:**
*   **Empty Tree:** If the root is null or empty, the result vector remains empty. [G]
*   **Degenerate Input:** If the tree structure is corrupted (e.g., branch pointing to null), the implementation must handle null checks. [G]

## Subtopics

### Leaf Node Representation
A specific variant focusing on the data layout of leaf nodes in the [zazen](https://github.com/gabrielmuller/zazen) implementation [S2, unofficial]. Leaves are represented as 4-byte RGBA blocks. A leaf is considered valid if `rgba[3] != 0x00` [S2, unofficial]. The `set_color(pixel, lightness)` function multiplies the RGB channels by `lightness` but leaves the alpha channel unchanged [S2, unofficial]. Comparison uses bitwise equality of the entire 4-byte block, not element-wise checks [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of nodes in the tree, as every node is visited exactly once. [G]
*   **Memory Complexity:** $O(L)$ where $L$ is the number of leaf/terminal nodes, for the result vector. [G]
*   **Strengths:** Simple to implement; decouples traversal from rendering; efficient for batch processing. [G]
*   **Limitations:** Requires storing all leaf nodes in memory; not suitable for ray-by-ray rendering where early termination is critical. [G]
*   **When to use:** When rendering requires all visible geometry (e.g., global illumination, shadow maps) or when parallelizing leaf processing. [G]
*   **When not to use:** For standard ray marching where early exit on empty space is desired. [G]

## Background
Leaf Collection is a fundamental technique in hierarchical volume rendering, often used as a preprocessing step. [G] The specific implementation details cited here are derived from open-source repositories [isomesh](https://github.com/narnia-ai-mason/isomesh) and [zazen](https://github.com/gabrielmuller/zazen). [S1, unofficial][S2, unofficial]

## Sources
[S1] isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S2] zazen, https://github.com/gabrielmuller/zazen, fetched 2026-10-26 (unofficial)
