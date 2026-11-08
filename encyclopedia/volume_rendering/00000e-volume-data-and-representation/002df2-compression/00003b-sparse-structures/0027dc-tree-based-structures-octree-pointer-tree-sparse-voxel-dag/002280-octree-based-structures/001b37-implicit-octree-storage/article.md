# Implicit Octree Storage
Also called: Implicit Order Storage
## Overview
Implicit Octree Storage is a memory-efficient serialization technique for sparse voxel octrees that eliminates explicit node pointers by sorting the tree nodes into a linear array using an implicit traversal order, such as breadth-first order [S1]. It solves the problem of high memory overhead associated with pointer-based tree structures, making it suitable for streaming or persistent storage of large volumetric datasets [S1]. Unlike [Implicit Octree Traversal](1220-implicit-octree-traversal.md), which focuses on GPU-side traversal algorithms, this method focuses on the data layout for storage, and unlike [OpenVDB (VDB)](1508-openvdb-vdb.md), which uses a hybrid structure with explicit pointers for interior nodes, it stores the entire hierarchy implicitly [S1]. The primary trade-off is that while it minimizes memory footprint, the resulting linear layout is not suitable for efficient random-access traversal without reconstruction [S1].

## How it works
**Inputs and Outputs:**
- Input: A sparse voxel octree data structure (typically defined by node types, child pointers, and scalar/vector attributes).
- Output: A contiguous linear array (or file stream) of node data, ordered according to the chosen implicit traversal sequence.

**Data Structures:**
The output is a flat array of node records. Each record must contain all necessary information to reconstruct the node's state, as no external pointers exist.
- **Node Record Layout:**
  - `type`: Enum or bit-field indicating node status (e.g., Empty, Interior, Leaf).
  - `attributes`: Scalar or vector data (e.g., density, color) if the node is a leaf.
  - `child_mask`: A bitmask (e.g., 8 bits for an octree) indicating which of the 8 potential children exist in the implicit sequence.
  - *Note:* The spatial position is implicit based on the traversal order and the parent-child relationships defined by the mask and array index.

**Algorithm:**
1. **Traversal Order Selection:** Choose an implicit ordering scheme. Breadth-first order (level-order) is explicitly cited as a viable method [S1]. Depth-first order is also a common implicit strategy, though not explicitly detailed in the provided sources for this specific method, breadth-first is the primary example given [S1].
2. **Linearization:** Perform a traversal of the octree using the selected order.
3. **Node Serialization:** For each node visited:
   a. Determine the node's type and attributes.
   b. Determine the existence of its 8 children.
   c. Write the node's data to the output array.
   d. Do *not* write child pointers. The children will appear later in the array at positions determined by the traversal logic.
4. **Completion:** The resulting array represents the entire tree. To access a specific node, one must traverse the array from the root, using the child masks to determine the offset of the next node in the sequence.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def serialize_octree_bfs(root_node):
    """
    Serializes an octree into a flat array using breadth-first order.
    Assumes a standard octree node structure with children[8] and attributes.
    """
    output_buffer = []
    queue = [root_node]
    
    while queue:
        current_node = queue.pop(0)
        
        # Serialize current node data
        node_record = {
            'type': current_node.type,
            'attributes': current_node.attributes,
            'child_mask': 0
        }
        
        # Determine which children exist and add them to the queue
        for i in range(8):
            if current_node.children[i] is not None:
                node_record['child_mask'] |= (1 << i)
                queue.append(current_node.children[i])
        
        output_buffer.append(node_record)
        
    return output_buffer
```

**Parameters:**
- **Ordering Scheme:** Breadth-first is explicitly mentioned [S1]. Other schemes (e.g., depth-first) are not detailed in the sources.
- **Compression:** The sources note that this storage format can be combined with lossless compression libraries like LZO for further size reduction, particularly in out-of-core implementations [S2].

**Edge Cases and Pitfalls:**
- **Traversal Efficiency:** The implicit order is explicitly noted as "not suitable for efficient traversal" [S1]. Random access requires re-traversing from the root or maintaining auxiliary index structures, which adds complexity.
- **Memory vs. Speed:** While memory size is reduced by removing pointers, the lack of spatial locality for traversal (compared to pointer-chasing in some cache-friendly layouts) can impact rendering performance if not handled by specific traversal algorithms [S1].

## Subtopics

### Out-of-Core Octree Storage
This variant stores the volume octree in an out-of-core structure, specifically using Berkeley DB for large dataset management [S2]. It integrates lossless compression using the LZO library to manage memory constraints for datasets that exceed available RAM [S2]. This approach is distinct from in-memory implicit storage by handling persistence and disk I/O directly through the database interface.

## Operating characteristics
- **Memory Complexity:** Significantly reduced compared to pointer-based octrees, as each node stores only a small bitmask and attributes instead of 8 pointers [S1]. The reduction factor depends on the pointer size and node density.
- **Time Complexity:** Serialization is linear with respect to the number of nodes $O(N)$, where $N$ is the number of non-empty nodes. Deserialization or traversal is also $O(N)$ but with higher constant factors due to the lack of direct pointer access [S1].
- **Performance:** Suitable for streaming and storage [S1]. Not suitable for real-time rendering traversal without additional acceleration structures or traversal algorithms designed for implicit layouts [S1].
- **Strengths:** Minimal memory footprint; simple linear data structure; compatible with standard compression libraries (e.g., LZO) [S1, S2].
- **Limitations:** Poor traversal performance for random access; requires full reconstruction or careful indexing for efficient rendering [S1].

## Background
The concept of storing octrees without pointers by sorting them in an implicit order (such as breadth-first) was highlighted in the context of reducing memory consumption for point clouds discretized into octrees [S1]. The work by Schnabel and Klein [2006] is cited as focusing on decreasing memory consumption for 4K resolution point clouds using this technique [S1]. The approach is related to common subtree merging techniques used in cartographic image compression [S1]. The out-of-core variant using Berkeley DB and LZO compression was demonstrated in the context of volumetric display of the human musculoskeletal system [S2].

## Sources
[S1] (PDF) High resolution sparse voxel DAGs - ResearchGate, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S2] Towards Advanced Volumetric Display of the Human Musculoskeletal System, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/021-025, fetched 2026-10-26
