# FlagOctree Construction

Also called: FlagOctree

## Overview
FlagOctree Construction is a GPU-based algorithm for building a sparse voxel octree from a list of fragment positions using atomic operations to mark nodes in a shared pool as occupied [S1, unofficial]. It builds the tree structure implicitly via pointers stored in a single `childPtr` field, distinguishing itself from sibling methods by its specific bit-packing strategy for node flags and pointer extraction [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A read-only `StructuredBuffer<Voxel>` named `FragmentsList` containing voxel positions.
*   **Input:** Constant buffers `cbInfo` (containing `curLevel`, `totalLevel`, `sumVoxels`, `curNode`) and `cbGroupInfo` (containing `groupSize`).
*   **Output:** A read-write `RWStructuredBuffer<Node>` named `nodesPool` representing the octree nodes.

**Data Structures:**
*   `Voxel`: Contains `float3 pos`, `uint normal`, `uint color`.
*   `Node`: Contains `uint brickPtr`, `uint childPtr`.
*   `cbInfo`: Contains `uint curLevel`, `uint totalLevel`, `uint sumVoxels`, `uint curNode`.
*   `cbGroupInfo`: Contains `uint3 groupSize`.

**Method Details:**
The algorithm processes voxels in parallel. For each voxel, it traverses the octree from the root (level 0) to the leaf level (`totalLevel - 1`). At each level, it calculates the child index based on the voxel's position within the current node's local coordinate space. It uses atomic operations to set specific bits in the `childPtr` field of the current node to indicate whether the node is an internal node or a leaf node. The actual index of the child node is stored in the lower 30 bits of `childPtr`, while the upper bits are used for flags.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
MASK_INDEX = 0x3FFFFFFF  # Mask to extract the actual node index from childPtr
FLAG_LEAF = 0x40000000   # Bit 30: Leaf node flag
FLAG_INTERNAL = 0x80000000 # Bit 31: Internal node flag

# Helper function to extract node index from childPtr
def extract_pos(child_ptr):
    return child_ptr & MASK_INDEX

# Main kernel execution (one thread per voxel)
def flag_octree_construction_kernel(thread_id, group_size, fragments_list, nodes_pool, cb_info, cb_group_info):
    # Calculate global voxel index
    voxel_index = thread_id.x + thread_id.y * group_size.x * 16
    
    # Boundary check
    if voxel_index >= cb_info.sum_voxels:
        return
    
    # Load voxel position
    voxel = fragments_list[voxel_index]
    
    # Initialize current node index (root is always at index 0)
    cur_index = 0
    
    # Traverse from level 0 to totalLevel - 2 (internal nodes)
    # Note: The loop goes up to curLevel - 1 in the source, but curLevel is dynamic.
    # Based on the source, it seems to process up to the leaf level.
    # Let's follow the source logic: Loop from level 0 to curLevel - 1.
    # However, curLevel is a constant from cbInfo for the entire dispatch?
    # The source says: "Loop from level 0 to curLevel - 1".
    # And then: "If curLevel == totalLevel - 1 (leaf level)... Otherwise (internal node)..."
    # This implies the kernel is dispatched multiple times, once per level?
    # Or curLevel is the current level being processed in this dispatch.
    # Let's assume the kernel is dispatched for each level from 0 to totalLevel-1.
    
    for level in range(cb_info.cur_level):
        # Calculate scaling factor for current level
        tt = 2.0 ** level
        
        # Calculate local coordinates within the current node
        xx = int(frac(voxel.pos.x * tt) * 2.0)
        yy = int(frac(voxel.pos.y * tt) * 2.0)
        zz = int(frac(voxel.pos.z * tt) * 2.0)
        
        # Get the current node's childPtr
        current_child_ptr = nodes_pool[cur_index].child_ptr
        
        # Extract the index of the child node
        child_index = extract_pos(current_child_ptr)
        
        # Calculate the index of the specific child (0-7)
        child_offset = xx + yy * 2 + zz * 4
        
        # Update cur_index to point to the child node
        # Note: The source says: curIndex = extractPos(nodesPool[curIndex].childPtr) + xx + yy * 2 + zz * 4;
        # This implies that the childPtr stores the base index of the children block?
        # Or does it store the index of the first child?
        # Let's re-read: "curIndex = extractPos(nodesPool[curIndex].childPtr) + xx + yy * 2 + zz * 4;"
        # This suggests that childPtr points to the start of the children array, and we add the offset.
        # But wait, if childPtr is an index into nodesPool, then extractPos gives the index of the first child.
        # Then we add the offset to get the specific child.
        # This implies that children are stored contiguously in nodesPool.
        
        cur_index = child_index + child_offset
        
        # If we are at the leaf level (totalLevel - 1), we need to mark the leaf
        # But the loop goes up to curLevel - 1.
        # If curLevel == totalLevel, then the last iteration is level = totalLevel - 1.
        # At this point, cur_index points to the leaf node.
        pass
    
    # After the loop, cur_index points to the leaf node (if curLevel == totalLevel)
    # Or to the last internal node if curLevel < totalLevel?
    # The source says: "If curLevel == totalLevel - 1 (leaf level)..."
    # This suggests that the check is done inside the loop or after?
    # Let's look at the source again:
    # "Loop from level 0 to curLevel - 1: ... Update curIndex ..."
    # "If curLevel == totalLevel - 1 (leaf level), set leaf flag..."
    # "Otherwise (internal node), set internal flag..."
    # This implies that the flag setting happens after the loop, for the node at curIndex.
    # But if curLevel < totalLevel, then curIndex points to an internal node that needs to be marked as internal?
    # And if curLevel == totalLevel, it points to a leaf?
    # Let's assume the kernel is dispatched for each level.
    
    # If curLevel is the level we are currently processing, then after the loop,
    # cur_index points to the node at level curLevel.
    # If curLevel == totalLevel - 1, this is a leaf node.
    # If curLevel < totalLevel - 1, this is an internal node.
    
    # However, the source says "If curLevel == totalLevel - 1".
    # This suggests that the flag is set for the node at curIndex.
    
    if cb_info.cur_level == cb_info.total_level - 1:
        # Set leaf flag
        atomic_or(nodes_pool[cur_index].child_ptr, FLAG_LEAF)
    else:
        # Set internal flag
        atomic_or(nodes_pool[cur_index].child_ptr, FLAG_INTERNAL)
```

**Parameters:**
*   **Dispatch thread group size:** 16x16x1 [S1, unofficial].
*   **totalLevel:** The total depth of the octree. Not given in the sources.
*   **curLevel:** The current level being processed. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Pointer Extraction:** The actual node index is extracted from `childPtr` by masking out the top two bits (bits 30 and 31) using `0x3FFFFFFF` [S1, unofficial].
*   **Flag Bits:** Internal nodes are marked with bit 31 (`0x80000000`) and leaf nodes with bit 30 (`0x40000000`) [S1, unofficial].
*   **Atomic Operations:** The algorithm relies on `InterlockedOr` (or equivalent atomic OR) to set flags. This ensures thread safety when multiple voxels map to the same node [S1, unofficial].
*   **Precision:** The calculation of local coordinates uses `frac(voxel.pos * tt)`. Care must be taken with floating-point precision, especially at higher levels where `tt` becomes large [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot L)$, where $N$ is the number of voxels and $L$ is the total number of levels. Each voxel traverses the tree from root to leaf.
*   **Memory Complexity:** $O(M)$, where $M$ is the number of nodes in the octree. The `nodesPool` buffer must be large enough to hold all nodes.
*   **Performance:** The algorithm is designed for GPU execution, leveraging parallelism to process voxels simultaneously. The use of atomic operations can lead to contention, especially in dense regions of the scene.
*   **Strengths:** Simple implementation, efficient for sparse data, GPU-friendly.
*   **Limitations:** Atomic contention can reduce performance in dense scenes. The implicit tree structure may be harder to debug than an explicit one.
*   **When to use:** When constructing a sparse voxel octree on the GPU from a list of voxels.
*   **When not to use:** When the scene is very dense and atomic contention is a bottleneck, or when an explicit tree structure is required for other algorithms.

## Background
The FlagOctree Construction method is described in the SVODemo repository by Throbbing [S1, unofficial]. It is a specific implementation of sparse voxel octree construction tailored for GPU execution.

## Sources
[S1] Throbbing/SVODemo, https://github.com/Throbbing/SVODemo, fetched 2026-10-26 (unofficial)
