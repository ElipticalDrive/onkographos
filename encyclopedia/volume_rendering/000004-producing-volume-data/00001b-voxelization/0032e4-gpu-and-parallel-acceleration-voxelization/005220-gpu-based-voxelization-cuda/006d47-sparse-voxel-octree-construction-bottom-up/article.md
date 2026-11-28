# Sparse Voxel Octree Construction (Bottom-Up)

Also called: Bottom-Up SVO, SVO Construction

## Overview
This method constructs a Sparse Voxel Octree (SVO) from a pre-voxelized surface representation using a bottom-up approach on the GPU. It groups compacted surface voxels by their Morton codes to identify parent-child relationships, creating internal nodes that represent occupied regions. It differs from top-down methods by starting with leaf voxels and merging them upwards, and from hash-table approaches by explicitly building the hierarchical tree structure rather than relying on dynamic hashing for traversal [S1, unofficial].

## How it works
**Inputs:** A list of surface voxels represented by their 32-bit Morton codes.
**Outputs:** A Sparse Voxel Octree structure containing internal nodes and leaf nodes.

**Data Structures:**
*   `d_pactDataArray`: Device array of compacted Morton codes.
*   `d_isValidArray`: Device array of booleans indicating valid voxels before compaction.
*   `d_esumDataArray`: Device array of exclusive sums used for compaction offsets.
*   `d_parentMortonArray`: Device array storing Morton codes of identified parent nodes.
*   `d_nNodesArray`: Device array storing node counts (8 for new parents, 0 for continuations).
*   `d_sumNodesArray`: Device array of prefix sums for node offsets.

**Parameters:**
*   `E_MORTON_32_FLAG`: Bit 31 (value $2^{31}$) used as a flag to indicate a valid node or surface voxel in the Morton code [S1, unofficial].

**Steps:**

1.  **Compact Surface Voxels:**
    Use a parallel kernel (`compactArray`) to gather valid voxels from `d_isValidArray` into `d_pactDataArray`. The destination index for each voxel is determined by its value in `d_esumDataArray` (exclusive sum) [S1, unofficial].

2.  **Identify Parent Nodes:**
    Iterate through the compacted array `d_pactDataArray`. For each voxel at index `tid > 0`, check if it shares a parent with the previous voxel (`tid - 1`). Two voxels share a parent if their parent Morton codes are equal [S1, unofficial].
    *   Helper: `getParentMorton(m)` returns `((m >> 3) & 0xfffffff)` [S1, unofficial].
    *   Helper: `isSameParent(m1, m2)` returns `getParentMorton(m1) == getParentMorton(m2)` [S1, unofficial].

3.  **Mark Parent Nodes:**
    If a new parent is identified (i.e., `tid == 0` or `!isSameParent(d_pactDataArray[tid], d_pactDataArray[tid - 1])`):
    *   Compute the parent's Morton code: `parentMorton = getParentMorton(d_pactDataArray[tid])`.
    *   Write `parentMorton` to `d_parentMortonArray` with the `E_MORTON_32_FLAG` set.
    *   Set `d_nNodesArray[tid] = 8` to represent the 8 children of this new parent.
    *   If the voxel continues the same parent as the previous one, set `d_nNodesArray[tid] = 0` [S1, unofficial].

4.  **Create Nodes:**
    Use the computed node counts in `d_nNodesArray` to build the final SVO structure. A prefix sum (`d_sumNodesArray`) is used to calculate the memory offsets for each node in the output structure. Iterate through the compacted data to populate the tree nodes based on these offsets [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper functions
def get_parent_morton(morton_code: uint32) -> uint32:
    """Returns the Morton code of the parent node."""
    return (morton_code >> 3) & 0xfffffff

def is_same_parent(m1: uint32, m2: uint32) -> bool:
    """Checks if two voxels share the same parent."""
    return get_parent_morton(m1) == get_parent_morton(m2)

# Step 1: Compact Array
# Kernel: compactArray
# Input: d_isValidArray (bool), d_pactDataArray (uint32), d_esumDataArray (uint32)
# Logic:
# tid = thread_id
# if d_isValidArray[tid]:
#     dest_idx = d_esumDataArray[tid]
#     d_pactDataArray[dest_idx] = original_morton_code[tid]

# Step 2 & 3: Identify and Mark Parents
# Kernel: cpNumNodes
# Input: d_pactDataArray (uint32), d_parentMortonArray (uint32), d_nNodesArray (uint32)
# Constants: E_MORTON_32_FLAG = 0x80000000
# Logic:
# tid = thread_id
# if tid > 0:
#     if not is_same_parent(d_pactDataArray[tid], d_pactDataArray[tid - 1]):
#         # New parent found
#         parent_m = get_parent_morton(d_pactDataArray[tid])
#         d_parentMortonArray[tid] = parent_m | E_MORTON_32_FLAG
#         d_nNodesArray[tid] = 8
#     else:
#         # Continuation of same parent
#         d_nNodesArray[tid] = 0
# else:
#     # First element, always a new parent
#     parent_m = get_parent_morton(d_pactDataArray[tid])
#     d_parentMortonArray[tid] = parent_m | E_MORTON_32_FLAG
#     d_nNodesArray[tid] = 8

# Step 4: Create Nodes
# Kernel: createNode_1
# Input: d_pactDataArray, d_nNodesArray, d_sumNodesArray (prefix sum of d_nNodesArray)
# Logic:
# Iterate through compacted data.
# Use d_sumNodesArray to determine offsets for writing nodes to the final SVO structure.
# Not given in the sources.
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of surface voxels for compaction and parent identification. Prefix sums add a logarithmic or linear overhead depending on implementation.
*   **Memory Complexity:** Requires additional device memory for intermediate arrays (`d_esumDataArray`, `d_parentMortonArray`, `d_nNodesArray`, `d_sumNodesArray`).
*   **Strengths:** Efficient on GPU due to parallel compaction and simple bitwise operations for Morton code manipulation. Avoids the overhead of dynamic hash table resizing.
*   **Limitations:** Requires pre-voxelized data. The bottom-up approach may create redundant nodes if the input voxel list is not sorted by Morton code (the algorithm assumes sorted input for `isSameParent` to work correctly across adjacent elements).
*   **When to use:** When a hierarchical SVO is required for ray marching or acceleration, and the input is already voxelized.
*   **When not to use:** If memory is extremely constrained and a flat hash table suffices, or if the input is not sorted.

## Background
This method is derived from GPU-based voxelization techniques, specifically those utilizing CUDA for surface voxelization. The bottom-up SVO construction is a common pattern in GPU ray tracing pipelines where a sparse hierarchy is needed for efficient traversal. The specific implementation details (Morton code manipulation, compaction kernels) are found in open-source GPU SVO construction projects [S1, unofficial].

## Sources
[S1] GitHub - Alan-Leo-Wong/GPU-SVO, https://github.com/Alan-Leo-Wong/GPU-SVO, fetched 2026-10-29 (unofficial)
