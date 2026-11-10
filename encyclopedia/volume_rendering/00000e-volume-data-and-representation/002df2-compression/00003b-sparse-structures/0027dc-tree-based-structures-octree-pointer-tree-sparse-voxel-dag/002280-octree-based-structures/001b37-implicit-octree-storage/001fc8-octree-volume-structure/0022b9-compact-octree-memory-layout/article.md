# Compact Octree Memory Layout

Also called: CNode Layout, Compact Voxel Storage

## Overview
Compact Octree Memory Layout is a memory layout for voxel octree nodes that packs validity and leaf status into 2 bytes and uses relative pointers, with support for far pointers via a separate indirection entry to handle large address spaces [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A logical octree structure defining the spatial hierarchy of volumetric data [S1, unofficial].
- Output: A linear memory buffer containing `CNode` structures and optional far-pointer indirection entries [S1, unofficial].

**Data Structures:**
The core unit is the `CNode` structure [S1, unofficial]. It contains the following fields [S1, unofficial]:
1. **`valid_mask`**: An 8-bit bitmask where bit $i$ corresponds to child $i$. A value of 1 means the child exists (is not empty), and 0 otherwise [S1, unofficial].
2. **`leaf_mask`**: An 8-bit bitmask where bit $i$ corresponds to child $i$. A value of 1 means the child is a leaf node (has no further children), and 0 otherwise [S1, unofficial].
3. **`child_ptr`**: An array of eight 16-bit relative offsets, one per child. If bit 15 is set, it indicates a far pointer, and the remaining 15 bits (masked) serve as an index to a far entry. The far entry contains a 32-bit `far_ptr` with the actual offset [S1, unofficial].
4. **`far_ptr`**: A 32-bit absolute or relative pointer stored in a separate `CNode` entry (or dedicated array) when `child_ptr` indicates a far reference [S1, unofficial].

**Parameters:**
- `_shortPtrTop`: $2^{15}$ (32768). Threshold for `child_ptr` to indicate a far pointer [S1, unofficial].
- `_longPtrTop`: $2^{32}$. Threshold for `far_ptr` size [S1, unofficial].

**Algorithm for Traversal/Access:**
To access the $i$-th child of a node [S1, unofficial]:
1. Check `valid_mask` bit $i$. If 0, the child does not exist [S1, unofficial].
2. Read `child_ptr[i]` [S1, unofficial].
3. If `child_ptr[i] < _shortPtrTop`: the child is located at `base_address + child_ptr[i]` [S1, unofficial].
4. If `child_ptr[i] >= _shortPtrTop`: extract the index `idx = child_ptr[i] & (_shortPtrTop - 1)`, retrieve the `far_ptr` value from the indirection entry at `idx`, and locate the child at `base_address + far_ptr` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
SHORT_PTR_TOP = 1 << 15
SHORT_PTR_MASK = SHORT_PTR_TOP - 1

class CNode:
    valid_mask: int      # 8-bit
    leaf_mask: int       # 8-bit
    child_ptrs: list[int] # 8 elements, each 16-bit

class FarPointerTable:
    entries: list[int]   # List of 32-bit integers

def get_child_address(node: CNode, child_index: int, base_address: int, far_table: FarPointerTable) -> int:
    if not (node.valid_mask & (1 << child_index)):
        return None
    ptr = node.child_ptrs[child_index]
    if ptr < SHORT_PTR_TOP:
        return base_address + ptr
    else:
        idx = ptr & SHORT_PTR_MASK
        far_offset = far_table.entries[idx]
        return base_address + far_offset

def is_leaf(node: CNode, child_index: int) -> bool:
    return bool(node.leaf_mask & (1 << child_index))
```

**Edge Cases and Pitfalls:**
- Relative offsets are limited to 15 bits for short pointers. If the distance between a parent and child exceeds $2^{15}$ bytes, a far pointer must be used [S1, unofficial].
- Empty subtrees are represented by setting the corresponding bit in `valid_mask` to 0, effectively skipping the child during traversal [S1, unofficial].

## Subtopics

### Residency Octree Data Layout
Implemented as a full, pointerless octree for GPU simplicity [S2]. Each node consists of $m$ unsigned 32-bit integers, where $m$ is the number of channels [S2]. For each channel within a node: 16 bits track partially mapped resolutions, 8 bits store the minimum scalar value, and 8 bits store the maximum scalar value in the region represented by the node [S2]. This layout is designed for multi-channel volume rendering on the GPU, avoiding pointer indirection entirely [S2].

### Octree Node Grouping & Serialization
Groups up to 8 child nodes into a single parent node record, storing a base offset to the first child and relative offsets for subsequent children [S3, unofficial]. Null children are marked with offset -1 [S3, unofficial]. The node record fields are `m_Data` (size_t), `m_ChildrenBase` (size_t), and `m_ChildrenOffset[8]` (char/int, where -1 indicates an empty child) [S3, unofficial]. To serialize, iterate $i$ from 0 to 7: if child $i$ is not null, set `m_ChildrenBase` to the current position for the first non-null child and set the offset to 0 for that child, otherwise set the offset to `m_NodePosition - m_ChildrenBase`; write the child node and increment `m_NodePosition`; if child $i$ is null, set the offset to -1 [S3, unofficial].

## Operating characteristics
Not given in the sources.

## Background
The Compact Octree Memory Layout (CNode) was introduced in the context of software volume ray tracing, specifically within the `svo-workbench` project [S1, unofficial].

## Sources
[S1] skyfex/svo-workbench, https://github.com/skyfex/svo-workbench, fetched 2026-10-26 (unofficial)
[S2] Residency Octree: A Hybrid Approach for Scalable Web ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC10840607/, fetched 2026-10-29
[S3] GilesPenfold/MasterThesisSVO, https://github.com/GilesPenfold/MasterThesisSVO, fetched 2026-10-26 (unofficial)
