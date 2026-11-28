# VDB Random Deletion

Also called: VDB Random Delete, Sparse Voxel Octree Node Deletion

## Overview
VDB Random Deletion is an algorithm for removing values from a Vector Displacement Buffer (VDB) sparse voxel octree, designed for efficiency with dynamic data [S1, unofficial]. It unsets bits in child and value masks and prunes empty nodes bottom-up [S1, unofficial]. It differs from [VDB Random Insert](22375-vdb-random-insert.md) by traversing to locate the target and then clearing masks rather than setting them, followed by a bottom-up cleanup phase that [VDB Random Insert] does not perform [S1, unofficial].

## How it works
**Inputs and outputs:**
- Input: A VDB tree root node, and a 3D coordinate or addressing key identifying the voxel to delete.
- Output: The modified VDB tree with the specified voxel removed and empty subtrees pruned.

**Data structures:**
The VDB node structure includes:
- `mChildMask`: A bitmask indicating which of the 8 children exist.
- `mValueMask`: A bitmask indicating which voxels within a leaf node contain active data.
- `mChild`: A pointer to an array of child nodes (if `mChildMask` is non-zero).

**Steps:**
1. **Traverse:** Traverse the octree from the root to the target leaf node, similar to [VDB Random Insert](22375-vdb-random-insert.md) [S1, unofficial].
2. **Unset Masks:** At the target leaf node, unset the bit in `mValueMask` corresponding to the voxel index [S1, unofficial].
3. **Prune Bottom-Up:** Return to the parent and check if the node is now empty. A node is empty if:
   - It has no children (`mChildMask == 0`) AND no active voxels (`mValueMask == 0`) [S1, unofficial].
   - If empty, unset the bit in the parent's `mChildMask` that pointed to this node [S1, unofficial].
   - Recursively repeat this check up the tree until a non-empty node is reached or the root is processed [S1, unofficial].
4. **Efficient Pruning:** Implement pruning by checking multiple bits in `mChildMask` and `mValueMask` simultaneously using 64-bit arithmetic to check 64 entries at once [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vdb_random_delete(root, key):
    # Traverse to find the leaf node and the path
    path = []
    current = root
    for depth in range(MAX_DEPTH):
        path.append(current)
        if current.is_leaf():
            break
        child_idx = get_child_index(key, depth)
        if not (current.mChildMask & (1 << child_idx)):
            return False # Node does not exist
        current = current.mChild[child_idx]
    
    # Unset the value bit in the leaf
    leaf = current
    voxel_idx = get_voxel_index(key, MAX_DEPTH)
    if not (leaf.mValueMask & (1 << voxel_idx)):
        return False # Value not present
    
    leaf.mValueMask &= ~(1 << voxel_idx)
    
    # Prune bottom-up
    for node in reversed(path):
        # Check if node is empty
        if node.mValueMask == 0 and node.mChildMask == 0:
            # This node is empty, remove it from parent
            # Note: The parent is the next node in the reversed path, 
            # except for the root which has no parent to update.
            pass # Implementation detail: update parent's child mask
        else:
            break # Stop pruning as this node is not empty
    
    return True
```

**Parameters:**
- `MAX_DEPTH`: The maximum depth of the octree. Not given in the sources.

**Edge cases:**
- Deleting a non-existent node: The traversal will fail to find the child bit set, and the function should return false [S1, unofficial].
- Precision: Standard floating-point comparisons for coordinates may lead to inconsistent results if not handled with a tolerance. Not given in the sources.

## Subtopics

### Sparse Voxel Octree Node Deletion
This variant, found in Octomap, deletes a node given an addressing key or a 3D point [S2]. It always deletes at the lowest level unless a specific `depth` is provided [S2]. If pruning is involved, pruned nodes at the specified `depth` are deleted as a whole, and inner nodes are expanded as needed [S2]. It is chosen when using the Octomap library structure [S2].

## Operating characteristics
**Time and memory complexity:**
- Time: Average constant-time complexity, amortized over multiple coherent operations [S1, unofficial].
- Memory: No additional memory allocation; nodes are deallocated during pruning [S1, unofficial].

**Measured performance:**
- Not given in the sources.

**Scaling:**
- Independent of the topology or resolution of the underlying dataset on average [S1, unofficial].

**Strengths and limitations:**
- Strengths: Efficient for dynamic data, constant-time access [S1, unofficial].
- Limitations: Slower than random lookup due to dynamic allocation overhead, though amortized [S1, unofficial].

**When to use:**
- Use for dynamic data where nodes are frequently added and removed [S1, unofficial].
- Do not use if the dataset is static, as simpler structures may be more efficient.

## Background
VDB was introduced by Ken Museth in 2013 [S1, unofficial]. The algorithm is part of the VDB library, which supports high-resolution sparse volumes with dynamic topology [S1, unofficial]. It is used in applications requiring efficient dynamic sparse voxel management, such as fluid simulation and animation [S1, unofficial].

## Sources
[S1] VDB: High-Resolution Sparse Volumes with Dynamic Topology (Academy Award Winner) [pdf], http://www.museth.org/Ken/Publications_files/Museth_TOG13.pdf, fetched 2026-10-26 (unofficial)
[S2] octomap: octomap::OcTree Class Reference, https://octomap.github.io/octomap/doc/classoctomap_1_1OcTree.html, fetched 2026-10-24
