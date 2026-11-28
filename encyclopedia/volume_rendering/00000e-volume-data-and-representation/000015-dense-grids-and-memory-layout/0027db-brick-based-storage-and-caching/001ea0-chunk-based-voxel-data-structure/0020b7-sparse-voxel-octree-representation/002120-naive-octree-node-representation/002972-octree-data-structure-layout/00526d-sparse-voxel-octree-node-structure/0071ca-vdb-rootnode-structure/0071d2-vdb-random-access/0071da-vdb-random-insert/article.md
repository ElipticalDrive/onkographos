# VDB Random Insert

Also called: VDB Insert, Sparse Voxel Octree Insert

## Overview
VDB Random Insert is the algorithm for inserting values into a Vector Displacement Buffer (VDB) sparse voxel octree, primarily used for grid initialization or dynamic data updates such as rebuilding a narrow band of a level set [S1, unofficial]. It differs from [VDB Random Access](22363-vdb-random-access.md) by dynamically allocating child nodes when traversing to a target voxel, whereas random access only traverses existing structures. This dynamic allocation ensures a low memory footprint for sparse data but introduces overhead compared to lookup, which is amortized over coherent insert operations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A VDB tree (starting from `RootNode`), a target voxel position (defined by a perfect key or coordinate), and a value to insert.
*   **Output:** The VDB tree with the new node structure and value set; the inserted value.

**Data Structures:**
*   **RootNode:** Contains a map of perfect keys to child nodes.
*   **Internal Nodes:** Contain `mChildMask` (bitmask indicating presence of children) and `mValueMask` (bitmask indicating presence of active values).
*   **LeafNode:** Contains the actual voxel data buffer and `mValueMask`.

**Steps:**
1.  **Traverse:** Start at the `RootNode`. Use fast bit operations to determine the path to the target voxel [S1, unofficial].
2.  **Check Child Mask:** For each level, check if the corresponding bit in `mChildMask` is set.
3.  **Allocate if Missing:** If the bit in `mChildMask` is off, allocate a new child node and insert it into the parent's structure [S1, unofficial].
4.  **Descend:** Move to the child node corresponding to the target bit.
5.  **Terminate:** Stop when a `LeafNode` is reached (which may have been newly constructed in step 3).
6.  **Set Value:** Set the voxel value in the desired temporal buffer of the `LeafNode` [S1, unofficial].
7.  **Update Mask:** Set the corresponding bit in the `LeafNode`'s `mValueMask` to mark the voxel as active [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vdb_random_insert(root_node, target_key, value):
    """
    Inserts a value into the VDB tree at the location specified by target_key.
    target_key is assumed to be a valid perfect key or coordinate convertible to a path.
    """
    current_node = root_node
    
    # Traverse from RootNode down to the LeafNode level
    # Assuming a fixed depth or determined by the key structure
    for level in range(root_node.depth, 0, -1):
        # Determine the child index from the target key for this level
        child_index = get_child_index(target_key, level)
        
        # Check if child exists using mChildMask
        if not (current_node.mChildMask & (1 << child_index)):
            # Allocate new child node if bit is off
            new_child = allocate_new_node()
            insert_child_into_parent(current_node, new_child, child_index)
        
        # Move to the child node
        current_node = get_child_node(current_node, child_index)
        
        # If we reached a LeafNode before the final level, handle appropriately
        # In standard VDB, internal nodes don't store values, only leaves do.
        # So we continue until we are at the parent of the leaf level.
        if isinstance(current_node, LeafNode):
            break

    # current_node is now the LeafNode (or the node at the leaf level)
    # Set the voxel value in the temporal buffer
    leaf_buffer = current_node.get_buffer()
    leaf_buffer[target_key.local_index] = value
    
    # Set the corresponding bit in mValueMask
    current_node.mValueMask |= (1 << target_key.local_index)
```

**Parameters:**
*   **Temporal Buffer:** The specific buffer within the LeafNode where the value is stored. Not given in the sources.
*   **Bit Operations:** Standard bitwise AND, OR, and shift operations.

**Edge Cases and Pitfalls:**
*   **Dynamic Allocation Overhead:** Random insert is slower than random lookup due to dynamic node allocation [S1, unofficial].
*   **Amortization:** The overhead is amortized over multiple coherent insert operations, resulting in average constant-time complexity [S1, unofficial].
*   **Memory Footprint:** Nodes below the RootNode are allocated only on insert, keeping memory footprint low for sparse data [S1, unofficial].

## Subtopics

### [VDB Random Deletion](22410-vdb-random-deletion.md)
VDB Random Deletion is an algorithm for removing values from a Vector Displacement Buffer (VDB) sparse voxel octree, designed for efficiency with dynamic data. It unsets bits in child and value masks and prunes empty nodes bottom-up. It differs from [VDB Random Insert](22375-vdb-random-insert.md) by traversing to locate the target and then clearing masks rather than setting them, followed by a bottom-up cleanup phase that [VDB Random Insert] does not perform.

## Operating characteristics
*   **Time Complexity:** Average time complexity is $O(1)$ due to amortization over coherent insert operations [S1, unofficial].
*   **Memory Complexity:** Low memory footprint for sparse volumetric data because nodes below the RootNode are allocated only on insert [S1, unofficial].
*   **Performance:** Slower than random lookup due to dynamic allocation overhead, but efficient for batch coherent inserts [S1, unofficial].
*   **Strengths:** Efficient for dynamic data updates and initialization; low memory usage for sparse data.
*   **Limitations:** Slower than lookup for single insertions; requires dynamic memory allocation.

## Background
VDB (Vector Displacement Buffer) was introduced by Ken Museth in the paper "VDB: High-Resolution Sparse Volumes with Dynamic Topology" (TOG 2013, Academy Award Winner) [S1, unofficial]. The random insert operation is a core component of VDB's ability to handle dynamic topology changes, such as narrow band rebuilding for level sets [S1, unofficial].

## Sources
[S1] VDB: High-Resolution Sparse Volumes with Dynamic Topology (Academy Award Winner) [pdf], http://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf, fetched 2026-10-26 (unofficial)
