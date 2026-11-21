# Octree Node Lookup by Voxel


## Overview
This method traverses a Sparse Voxel Octree (SVO) from the root to locate the leaf node corresponding to a specific 3D integer voxel coordinate, returning the index of the payload associated with that leaf. It solves the problem of mapping dense voxel grid coordinates to sparse tree payloads, which is a prerequisite for operations like trilinear sampling or attribute retrieval. It differs from its sibling, [Octree Voxel Color Retrieval](5015-octree-voxel-color-retrieval.md), by returning only the payload index rather than the interpolated color value itself.

## How it works
**Inputs:**
- `nodes`: Array of `NodeDescriptor` structures representing the octree nodes.
- `num_nodes`: Number of nodes in the `nodes` array.
- `leaf_payload_indices`: Array of indices mapping leaf IDs to payload data.
- `num_leaves`: Number of leaves in the octree.
- `max_depth`: The maximum depth of the octree.
- `voxel`: A 3D integer coordinate $(x, y, z)$ to look up.

**Outputs:**
- An integer representing the payload index for the voxel. Returns $-1$ if the voxel is out of bounds, the path does not exist, or the leaf is invalid.

**Data Structures:**
The `NodeDescriptor` contains the following fields:
- `child_mask`: A bitmask indicating which of the 8 children exist (bit $i$ set if child $i$ exists).
- `leaf_mask`: A bitmask indicating which existing children are leaf nodes (subset of `child_mask`).
- `payload_base`: An offset into `leaf_payload_indices` for the first leaf child.
- `internal_child_mask`: A bitmask indicating which existing children are internal nodes (subset of `child_mask`).
- `child_base`: An offset into the `nodes` array for the first internal child.

**Algorithm:**
1. **Empty Check:** If `num_nodes` is 0, return $-1$.
2. **Depth 0 Check:** If `max_depth` is 0, check if a root leaf exists. If the root is a leaf and valid, return its payload index; otherwise, return $-1$.
3. **Bounds Check:** Calculate `grid_size` as $2^{\text{max\_depth}}$. If the voxel coordinate $(x, y, z)$ is not within the range $[0, \text{grid\_size})$, return $-1$.
4. **Initialization:** Set `node_index` = 0 (root node index) and `depth_remaining` = `max_depth`.
5. **Traversal Loop:**
   a. If `node_index` >= `num_nodes`, return $-1$ (invalid node reference).
   b. Retrieve the `NodeDescriptor` at `nodes[node_index]`.
   c. Calculate the `child_index` for the current depth using bit extraction from the voxel coordinate:
      $$ \text{child\_index} = ((x \gg \text{depth\_remaining}) \& 1) \mid ((y \gg \text{depth\_remaining}) \& 1) \ll 1 \mid ((z \gg \text{depth\_remaining}) \& 1) \ll 2 $$
   d. Calculate `child_bit` as $1 \ll \text{child\_index}$.
   e. **Existence Check:** If `(descriptor.child_mask() & child_bit) == 0`, the child does not exist. Return $-1$.
   f. **Leaf Check:** If `(descriptor.leaf_mask() & child_bit) != 0`, the child is a leaf.
      i. Calculate the leaf ID relative to the node's leaves: `leaf_id` = `prefix_rank(descriptor.leaf_mask(), child_index)`.
      ii. Calculate the global leaf index: `global_leaf_index` = `descriptor.payload_base()` + `leaf_id`.
      iii. Retrieve `payload_index` from `leaf_payload_indices[global_leaf_index]`.
      iv. Validate `payload_index`. If valid, return it. If invalid, return $-1$.
   g. **Internal Node Check:** Otherwise, the child is an internal node.
      i. Calculate the new node index: `new_node_index` = `descriptor.child_base()` + `prefix_rank(descriptor.internal_child_mask(), child_index)`.
      ii. Decrement `depth_remaining` by 1.
      iii. If `depth_remaining` <= 0, return $-1$ (reached max depth without finding a leaf).
      iv. Update `node_index` = `new_node_index` and continue the loop.

**Helper Function: `prefix_rank`**
Calculates the number of set bits in `mask` that are strictly less significant than `child_index`.
$$ \text{prefix\_rank}(\text{mask}, \text{child\_index}) = \text{popcount}(\text{mask} \& ((1 \ll \text{child\_index}) - 1)) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lookup_payload_index_by_voxel(nodes, num_nodes, leaf_payload_indices, num_leaves, max_depth, voxel):
    x, y, z = voxel
    
    # 1. Empty Check
    if num_nodes == 0:
        return -1
    
    # 2. Depth 0 Check
    if max_depth == 0:
        # Assuming root is always a leaf if max_depth is 0 and nodes exist
        # Check if root payload is valid
        if num_leaves > 0:
            return leaf_payload_indices[0]
        else:
            return -1
            
    # 3. Bounds Check
    grid_size = 1 << max_depth
    if not (0 <= x < grid_size and 0 <= y < grid_size and 0 <= z < grid_size):
        return -1
        
    node_index = 0
    depth_remaining = max_depth
    
    while True:
        # 5a. Invalid Node Check
        if node_index >= num_nodes:
            return -1
            
        descriptor = nodes[node_index]
        
        # 5c. Calculate Child Index
        child_depth = depth_remaining - 1
        x_bit = (x >> child_depth) & 1
        y_bit = (y >> child_depth) & 1
        z_bit = (z >> child_depth) & 1
        child_index = x_bit | (y_bit << 1) | (z_bit << 2)
        
        child_bit = 1 << child_index
        
        # 5e. Existence Check
        if (descriptor.child_mask & child_bit) == 0:
            return -1
            
        # 5f. Leaf Check
        if (descriptor.leaf_mask & child_bit) != 0:
            # Calculate rank among leaves
            lower_mask = (1 << child_index) - 1
            leaf_rank = popcount(descriptor.leaf_mask & lower_mask)
            
            global_leaf_index = descriptor.payload_base + leaf_rank
            
            if global_leaf_index < num_leaves:
                payload_index = leaf_payload_indices[global_leaf_index]
                # Validate payload index (e.g., check if it's not a sentinel value like -1 or UINT32_MAX)
                # [G] Assuming valid payload indices are non-negative integers.
                if payload_index >= 0:
                    return payload_index
            return -1
            
        # 5g. Internal Node Check
        # Calculate rank among internal children
        lower_mask = (1 << child_index) - 1
        internal_rank = popcount(descriptor.internal_child_mask & lower_mask)
        
        new_node_index = descriptor.child_base + internal_rank
        depth_remaining -= 1
        
        if depth_remaining <= 0:
            return -1
            
        node_index = new_node_index
```

## Subtopics

### [WriteLeafOctree](16593-writeleafoctree.md)
WriteLeafOctree is a compute shader that traverses a Sparse Voxel Octree (SVO) to locate the leaf node corresponding to each input voxel, then maps the voxel's attributes into a 3D brick pool texture. It differs from its parent, [Octree Node Lookup by Voxel](5120-octree-node-lookup-by-voxel.md), by not just returning the leaf index but actively populating a structured brick-based texture memory with the voxel data, effectively bridging the gap between sparse tree representation and dense brick-based rendering buffers.

## Operating characteristics
- **Time Complexity:** $O(D)$, where $D$ is the maximum depth of the octree. Each step involves constant-time bit operations and array lookups.
- **Memory Complexity:** $O(1)$ auxiliary space. The traversal uses a fixed number of registers/stack variables.
- **Performance:** The method is highly efficient for random access lookups in sparse data. Performance is limited by memory latency when accessing `nodes` and `leaf_payload_indices` arrays, which may be scattered in memory.
- **Strengths:** Exact mapping from voxel coordinates to payloads; handles sparse data efficiently; no interpolation overhead.
- **Limitations:** Does not handle partial overlaps (sub-voxel queries); requires exact integer coordinates; returns $-1$ for empty space, requiring caller handling.
- **When to use:** When exact payload retrieval for a specific voxel is needed, e.g., for building lookup tables or initializing dense buffers.
- **When not to use:** For trilinear sampling (use [Octree Voxel Color Retrieval](5015-octree-voxel-color-retrieval.md)); for dense grid access where a simple array lookup suffices.

## Background
The method is derived from implementations of Sparse Voxel Octrees used in modern ray tracing and volume rendering pipelines. The specific implementation details, including the `NodeDescriptor` layout and traversal logic, are found in the `hectorpiteau/sparse-voxels` repository [S1, unofficial]. The use of prefix ranks on child masks allows for compact storage of tree topology without explicit child pointers, a common optimization in SVOs.

## Sources
[S1] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
