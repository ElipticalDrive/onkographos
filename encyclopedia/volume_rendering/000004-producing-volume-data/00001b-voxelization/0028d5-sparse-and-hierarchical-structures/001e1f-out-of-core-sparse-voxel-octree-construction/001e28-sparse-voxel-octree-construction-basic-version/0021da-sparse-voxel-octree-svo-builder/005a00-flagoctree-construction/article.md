# FlagOctree Construction

Also called: FlagOctree

## Overview
FlagOctree Construction is a GPU-based algorithm for building a sparse voxel octree from a list of fragment positions [S1, unofficial]. It uses atomic operations to mark nodes in a shared pool as occupied and builds the tree structure implicitly via pointers stored in a single `childPtr` field [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A read-only `StructuredBuffer<Voxel>` named `FragmentsList` containing voxel positions [S1, unofficial].
*   **Input:** Constant buffers `cbInfo` (containing `curLevel`, `totalLevel`, `sumVoxels`, `curNode`) and `cbGroupInfo` (containing `groupSize`) [S1, unofficial].
*   **Output:** A read-write `RWStructuredBuffer<Node>` named `nodesPool` representing the octree nodes [S1, unofficial].

**Data Structures:**
*   `Voxel`: Contains `float3 pos`, `uint normal`, `uint color` [S1, unofficial].
*   `Node`: Contains `uint brickPtr`, `uint childPtr` [S1, unofficial].
*   `cbInfo`: Contains `uint curLevel`, `uint totalLevel`, `uint sumVoxels`, `uint curNode` [S1, unofficial].
*   `cbGroupInfo`: Contains `uint3 groupSize` [S1, unofficial].

**Method Details:**
The algorithm processes voxels in parallel using a dispatch thread group size of 16x16x1 [S1, unofficial]. For each voxel, it traverses the octree from the root (level 0) to the node at `curLevel` [S1, unofficial]. At each level, it calculates the child index based on the voxel's position within the current node's local coordinate space [S1, unofficial]. It uses atomic operations to set specific bits in the `childPtr` field of the current node to indicate whether the node is an internal node or a leaf node [S1, unofficial]. The actual index of the child node is stored in the lower 30 bits of `childPtr`, while the upper bits are used for flags [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
MASK_INDEX = 0x3FFFFFFF  # Mask to extract the actual node index from childPtr
FLAG_LEAF = 0x40000000   # Bit 30: Leaf node flag
FLAG_INTERNAL = 0x80000000 # Bit 31: Internal node flag

def frac(x):
    return x - math.floor(x)

def extract_pos(child_ptr):
    return child_ptr & MASK_INDEX

def InterlockedOr(ptr, flag):
    ptr.value |= flag

def flag_octree_construction_kernel(thread_id, group_size, fragments_list, nodes_pool, cb_info):
    # Calculate global voxel index
    voxel_index = thread_id.x + thread_id.y * group_size.x * 16
    
    # Boundary check
    if voxel_index >= cb_info.sum_voxels:
        return
    
    # Load voxel position
    voxel = fragments_list[voxel_index]
    
    # Initialize current node index (root is always at index 0)
    cur_index = 0
    
    # Traverse from level 0 to curLevel - 1
    for level in range(cb_info.cur_level):
        # Calculate scaling factor for current level
        tt = 2.0 ** level
        
        # Calculate local coordinates within the current node
        xx = int(frac(voxel.pos.x * tt) * 2.0)
        yy = int(frac(voxel.pos.y * tt) * 2.0)
        zz = int(frac(voxel.pos.z * tt) * 2.0)
        
        # Get the current node's childPtr and extract base index
        current_child_ptr = nodes_pool[cur_index].child_ptr
        child_index = extract_pos(current_child_ptr)
        
        # Calculate the index of the specific child (0-7)
        child_offset = xx + yy * 2 + zz * 4
        
        # Update cur_index to point to the child node
        cur_index = child_index + child_offset
    
    # Set flag for the node at cur_index
    if cb_info.cur_level == cb_info.total_level - 1:
        InterlockedOr(nodes_pool[cur_index], FLAG_LEAF)
    else:
        InterlockedOr(nodes_pool[cur_index], FLAG_INTERNAL)
```

**Parameters:**
*   **Dispatch thread group size:** 16x16x1 [S1, unofficial].
*   **totalLevel:** The total depth of the octree, stored in `cbInfo` [S1, unofficial].
*   **curLevel:** The current level being processed, stored in `cbInfo` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Pointer Extraction:** The actual node index is extracted from `childPtr` by masking out the top two bits (bits 30 and 31) using `0x3FFFFFFF` [S1, unofficial].
*   **Flag Bits:** Internal nodes are marked with bit 31 (`0x80000000`) and leaf nodes with bit 30 (`0x40000000`) [S1, unofficial].
*   **Atomic Operations:** The algorithm relies on `InterlockedOr` (or equivalent atomic OR) to set flags, ensuring thread safety when multiple voxels map to the same node [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
The FlagOctree Construction method is described in an unofficial repository by Throbbing [S1, unofficial]. It is a specific implementation of sparse voxel octree construction tailored for GPU execution [S1, unofficial]. Its origin, history, and development details are not given in the sources.

## Sources
[S1] Throbbing/SVODemo, https://github.com/Throbbing/SVODemo, fetched 2026-10-26 (unofficial)
