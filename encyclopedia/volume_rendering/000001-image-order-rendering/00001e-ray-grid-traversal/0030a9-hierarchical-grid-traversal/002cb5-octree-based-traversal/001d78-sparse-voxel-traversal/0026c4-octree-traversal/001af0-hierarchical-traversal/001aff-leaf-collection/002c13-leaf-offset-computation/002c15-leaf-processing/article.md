# Leaf Processing

Also called: IndexGridBuilder::foreach

## Overview
Leaf Processing is the final stage of the leaf collection pipeline, responsible for parallelizing the population of leaf node buffers with voxel data. It solves the problem of efficiently transferring precomputed and offset-adjusted data from a contiguous source buffer into the scattered memory layout of an OpenVDB tree's leaf nodes. It is distinguished from its parent, Leaf Offset Computation, by executing the actual data movement rather than calculating the destination addresses.

## How it works
**Inputs and Outputs:**
*   **Input:** A contiguous source buffer containing voxel values (`data`), a precomputed array of linear offsets for each leaf (`m_leafOffsets`), and a `LeafManager` instance managing the target leaf nodes.
*   **Output:** The leaf node buffers are populated with the corresponding voxel data.

**Data Structures:**
*   `m_leafOffsets`: An array of `size_t` where the element at index `leafIdx` holds the starting position in the source `data` array for that specific leaf.
*   `LeafT::SIZE`: The constant number of voxels in a leaf node (typically $8^3 = 512$ for standard OpenVDB leaves).
*   `ValueT`: The data type of the voxel values (e.g., `float`, `Vec3f`).

**Steps:**
1.  Initialize parallel iteration over the leaf nodes using `LeafManager::foreach`.
2.  Configure the parallel execution with a chunk size of 12 to balance overhead and granularity.
3.  For each leaf `leaf` in the current chunk:
    a. Retrieve the leaf's index `leafIdx`.
    b. Look up the base offset: `leafBaseOffset = m_leafOffsets[leafIdx]`.
    c. Retrieve the pointer to the leaf's internal buffer: `leafData = leaf.buffer().data()`.
    d. Copy the leaf's data from the source array to the leaf buffer using `std::memcpy`. The source address is `data + leafBaseOffset`, and the byte count is `LeafT::SIZE * sizeof(ValueT)`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def leaf_processing(data: List[ValueT], m_leafOffsets: List[int], leaf_manager: LeafManager, ValueT: type, LeafT: type):
    """
    Processes leaf nodes in parallel, copying data from a source buffer to leaf buffers using precomputed offsets.
    
    Args:
        data: Contiguous source buffer of voxel values.
        m_leafOffsets: Array of linear offsets for each leaf node.
        leaf_manager: Manager providing parallel iteration over leaves.
        ValueT: The data type of the voxels.
        LeafT: The leaf node type, providing SIZE constant.
    """
    chunk_size = 12
    
    # Parallel iteration over leaves
    # The lambda function is executed for each leaf in parallel chunks
    leaf_manager.foreach(
        lambda leaf: process_single_leaf(leaf, data, m_leafOffsets, ValueT, LeafT),
        parallel=True,
        chunk_size=chunk_size
    )

def process_single_leaf(leaf, data: List[ValueT], m_leafOffsets: List[int], ValueT: type, LeafT: type):
    """
    Copies data for a single leaf node from the source buffer to the leaf buffer.
    """
    # Get the index of the current leaf
    leaf_idx = leaf.index()
    
    # Get the base offset for the current leaf from the precomputed offsets array
    leaf_base_offset = m_leafOffsets[leaf_idx]
    
    # Get the leaf's buffer data pointer
    leaf_data = leaf.buffer().data()
    
    # Calculate the number of bytes to copy
    num_bytes = LeafT.SIZE * sizeof(ValueT)
    
    # Copy data from the source array to the leaf buffer
    # Source address is data + leaf_base_offset
    # Destination address is leaf_data
    memcpy(leaf_data, data[leaf_base_offset : leaf_base_offset + LeafT.SIZE], num_bytes)

def sizeof(type):
    """Returns the size in bytes of the given type."""
    if type == float:
        return 4
    elif type == double:
        return 8
    elif type == int:
        return 4
    elif type == Vec3f:
        return 12
    else:
        raise ValueError(f"Unknown type size for {type}")

def memcpy(dest, src, num_bytes):
    """
    Copies num_bytes from src to dest.
    In Python, this is simulated by list slicing and assignment or array operations.
    For the purpose of this algorithm description, we assume a direct memory copy operation.
    """
    # In a real C++ implementation, this would be std::memcpy
    # Here we simulate the data transfer
    for i in range(num_bytes):
        dest[i] = src[i]
```

**Parameters:**
*   `chunk_size`: Set to 12. This value balances thread overhead with the granularity of work distribution.

**Edge Cases and Pitfalls:**
*   **Offset Bounds:** If `m_leafOffsets` is incorrect or `leafIdx` is out of bounds, the copy will access invalid memory. The implementation assumes `m_leafOffsets` is correctly sized and populated by the parent step.
*   **Data Type Size:** `sizeof(ValueT)` must be accurate. Mismatched sizes will cause buffer overruns or incomplete copies.
*   **Parallel Safety:** The `leaf_manager.foreach` ensures that each leaf is processed by only one thread, preventing race conditions on the leaf buffers. The source `data` array is read-only, so no synchronization is needed for it.

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of active leaves. The work is linear in the number of leaves, with constant-time copy operations per leaf.
*   **Memory Complexity:** $O(1)$ additional memory per leaf, as the copy is done in-place into the leaf buffers. The source buffer and offset array are inputs.
*   **Performance:** The use of `std::memcpy` and parallel execution makes this step highly efficient. The chunk size of 12 is tuned for typical CPU core counts and cache line sizes.
*   **Strengths:** Simple, fast, and parallelizable. Leverages standard memory copy optimizations.
*   **Limitations:** Requires the source data to be contiguous and pre-offset. Does not handle data transformation or filtering.

## Background
This method is part of the HNanoSolver project, an unofficial implementation of sparse voxel grid processing. It is used in the context of building index grids for efficient ray tracing or volume rendering by parallelizing the population of leaf nodes.

## Sources
[S1] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
