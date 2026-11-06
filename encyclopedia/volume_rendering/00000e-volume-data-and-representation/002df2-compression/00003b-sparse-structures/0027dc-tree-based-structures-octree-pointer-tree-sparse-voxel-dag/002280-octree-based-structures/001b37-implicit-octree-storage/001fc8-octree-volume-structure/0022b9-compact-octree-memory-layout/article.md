# Compact Octree Memory Layout

Also called: CNode Layout, Compact Voxel Storage

## Overview
Compact Octree Memory Layout is a memory-efficient storage format for voxel octrees that packs node validity and leaf status into compact bitmasks while using relative pointers to minimize address space usage [S1, unofficial]. It solves the problem of high memory overhead in traditional pointer-based octrees by embedding structural metadata directly within node structures and deferring large address offsets to separate indirection entries [S1, unofficial]. Unlike sibling methods that may use explicit parent pointers or full-resolution grids, this layout optimizes for cache locality and reduced pointer size, making it suitable for large-scale volume rendering where memory bandwidth is a constraint [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A logical octree structure defining the spatial hierarchy of volumetric data.
- Output: A linear memory buffer containing `CNode` structures and optional `FarPointer` entries.

**Data Structures:**
The core unit is the `CNode` structure. It contains the following fields:

1.  **`valid_mask`**: An 8-bit unsigned integer. Bit $i$ (where $i \in [0, 7]$) is set to 1 if child $i$ exists (is not empty), and 0 otherwise.
2.  **`leaf_mask`**: An 8-bit unsigned integer. Bit $i$ is set to 1 if child $i$ is a leaf node (has no further children), and 0 otherwise.
3.  **`child_ptr`**: A 16-bit unsigned integer representing a relative offset to the child node.
    - If bit 15 of `child_ptr` is 0, the value is a direct relative offset to the child node within the memory buffer.
    - If bit 15 of `child_ptr` is 1, it indicates a "far pointer." The remaining 15 bits serve as an index into a separate indirection table.
4.  **`far_ptr`**: A 32-bit unsigned integer stored in a separate `CNode` entry (or a dedicated far-pointer array) when `child_ptr` indicates a far reference. This field holds the actual absolute or large relative offset to the target node.

**Parameters:**
- `_shortPtrTop`: $2^{15}$ (32768). The threshold value for `child_ptr`. If `child_ptr >= _shortPtrTop`, it is treated as a far pointer indicator [S1, unofficial].
- `_longPtrTop`: $2^{32}$. The threshold for the maximum size of a far pointer offset [S1, unofficial].

**Algorithm for Traversal/Access:**
To access the $i$-th child of a node:

1.  Check `valid_mask` bit $i$. If 0, the child does not exist.
2.  Read `child_ptr`.
3.  If `child_ptr < _shortPtrTop`:
    - The child is located at `base_address + child_ptr`.
4.  If `child_ptr >= _shortPtrTop`:
    - Extract the index: `idx = child_ptr & (_shortPtrTop - 1)`.
    - Retrieve the `far_ptr` value from the indirection entry at `idx`.
    - The child is located at `base_address + far_ptr`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
SHORT_PTR_TOP = 1 << 15
SHORT_PTR_MASK = SHORT_PTR_TOP - 1

class CNode:
    valid_mask: int  # 8-bit
    leaf_mask: int   # 8-bit
    child_ptr: int   # 16-bit
    # far_ptr is not stored in the main CNode if it's a far pointer,
    # but accessed via an indirection table.

class FarPointerTable:
    entries: list[int] # List of 32-bit integers

def get_child_address(node: CNode, child_index: int, base_address: int, far_table: FarPointerTable) -> int:
    """
    Calculate the memory address of the i-th child of a node.
    """
    # Check if child exists
    if not (node.valid_mask & (1 << child_index)):
        return None

    ptr = node.child_ptr
    
    if ptr < SHORT_PTR_TOP:
        # Short pointer: direct relative offset
        return base_address + ptr
    else:
        # Far pointer: index into indirection table
        idx = ptr & SHORT_PTR_MASK
        far_offset = far_table.entries[idx]
        return base_address + far_offset

def is_leaf(node: CNode, child_index: int) -> bool:
    """
    Check if the i-th child is a leaf node.
    """
    return bool(node.leaf_mask & (1 << child_index))
```

**Edge Cases and Pitfalls:**
- **Precision:** Relative offsets are limited to 15 bits for short pointers. If the distance between a parent and child exceeds $2^{15}$ bytes, a far pointer must be used [S1, unofficial].
- **Boundary Conditions:** The `valid_mask` and `leaf_mask` must be consistent. If a child is marked as a leaf in `leaf_mask`, it must also be marked as valid in `valid_mask` [S1, unofficial].
- **Degenerate Input:** Empty subtrees are represented by setting the corresponding bit in `valid_mask` to 0, effectively skipping the child during traversal [S1, unofficial].

## Subtopics

### Residency Octree Data Layout
Implemented as a full, pointerless octree for GPU simplicity [S2]. Each node consists of $m$ unsigned 32-bit integers, where $m$ is the number of channels [S2]. For each channel within a node: 16 bits track partially mapped resolutions, 8 bits store the minimum scalar value, and 8 bits store the maximum scalar value in the region represented by the node [S2]. This layout is designed for multi-channel volume rendering on the GPU, avoiding pointer indirection entirely [S2].

## Operating characteristics
- **Time Complexity:** Traversal is $O(depth)$ where depth is the height of the octree. Accessing a child is $O(1)$ for short pointers and $O(1)$ for far pointers (assuming the far pointer table is in cache).
- **Memory Complexity:** Significantly reduced compared to traditional pointer-based octrees due to the use of 16-bit relative offsets and packed bitmasks. However, far pointers introduce an indirection table that may increase memory usage for very large address spaces [S1, unofficial].
- **Performance:** Optimized for cache locality due to compact node size and linear memory layout [S1, unofficial]. Far pointers may cause cache misses if the indirection table is large [S1, unofficial].
- **Strengths:** Low memory overhead, efficient traversal for sparse volumes, suitable for large-scale rendering [S1, unofficial].
- **Limitations:** Limited address space for short pointers; far pointers add complexity and potential cache misses [S1, unofficial]. Not suitable for dynamic structures requiring frequent reallocation due to relative pointer invalidation [S1, unofficial].

## Background
The Compact Octree Memory Layout (CNode) was introduced in the context of software volume ray tracing, specifically within the `svo-workbench` project [S1, unofficial]. It was designed to provide a memory-efficient alternative to traditional octree storage formats for large-scale volumetric data [S1, unofficial]. The layout emphasizes compactness and relative addressing to reduce memory bandwidth requirements during ray traversal [S1, unofficial].

## Sources
[S1] skyfex/svo-workbench, https://github.com/skyfex/svo-workbench, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
