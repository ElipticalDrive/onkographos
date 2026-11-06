# Implicit Octree Storage
Also called: Implicit Order Storage
## Overview
Implicit Octree Storage is a serialization method for sparse voxel octrees that eliminates explicit node pointers by sorting tree nodes into a linear array using a deterministic spatial order, such as breadth-first order [S1]. It solves the problem of high memory overhead and fragmentation in pointer-based tree structures, making it suitable for streaming and persistent storage [S1]. Unlike traversal-optimized variants like [Implicit Octree Traversal](1220-implicit-octree-traversal.md), this method prioritizes compact storage over random-access speed, as the implicit ordering is explicitly noted as unsuitable for efficient traversal [S1].
## How it works
**Inputs and Outputs:**
- Input: A sparse voxel octree structure containing nodes and their parent-child relationships [S1].
- Output: A linear array (or stream) of node data, where the position in the array implicitly defines the parent-child relationships [S1].

**Data Structures:**
The output is a contiguous array of node records [S1]. Each record stores the necessary attribute data for the octree node (e.g., scalar values, flags, or bounding information) [S1]. The exact layout and size of each node record are Not given in the sources.

**Steps:**
1. **Traverse the Octree:** Perform a traversal of the input octree using the chosen implicit ordering algorithm (e.g., Breadth-First Search (BFS) or Morton Z-order) [S1, G].
2. **Serialize Nodes:** For each node visited during traversal, extract its attribute data [G].
3. **Store in Linear Array:** Append the serialized node data to the output array at the index corresponding to the node's position in the traversal order [G].
4. **Handle Sparsity:** Empty subtrees are omitted from the array, relying on the implicit ordering to reconstruct hierarchy during reconstruction or by storing explicit markers if required by the implementation [G].

**Formulas:**
- BFS Indexing (for a complete octree): Not given in the sources. Schnabel and Klein [S1] specify "breadth-first order" without detailing the exact arithmetic for child index calculation.
- Morton Code for 3D coordinate $(x, y, z)$: $M(x, y, z) = \text{interleave bits}(x, y, z)$ [G]. This maps 3D coordinates to a 1D integer key preserving spatial locality [G].

**Parameters:**
- Ordering Type: BFS or Morton (Z-order) [S1, G].
- Node Size: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Traversal Efficiency:** The implicit order is not suitable for efficient traversal [S1].
- **Sparsity Handling:** If the tree is highly sparse, a fixed-index BFS array may contain many null entries, wasting space. Morton order is more compact for sparse data but requires sorting [G].
- **Precision:** Morton codes can overflow for very large resolutions (e.g., >1024^3) if not using 64-bit integers [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def implicit_octree_storage(root_node, ordering='bfs'):
    """
    Serializes an octree into a linear array using implicit ordering.
    """
    nodes = []
    
    if ordering == 'bfs':
        queue = [root_node]
        while queue:
            current = queue.pop(0)
            nodes.append(current.serialize())
            if current.has_children:
                for child in current.children:
                    queue.append(child)
    elif ordering == 'morton':
        node_list = []
        collect_nodes_with_morton(root_node, node_list, depth=0, offset=(0, 0, 0))
        node_list.sort(key=lambda x: x['morton_code'])
        for item in node_list:
            nodes.append(item['node'].serialize())
    return nodes

def collect_nodes_with_morton(node, node_list, depth, offset):
    """
    Helper to collect nodes and compute Morton codes.
    """
    # Compute Morton code by interleaving bits of the 3D coordinates [G]
    m_code = interleave_bits(offset[0], offset[1], offset[2])
    node_list.append({'node': node, 'morton_code': m_code})
    
    if node.has_children:
        for i, child in enumerate(node.children):
            # Update offset for child octant
            new_offset = list(offset)
            for axis in range(3):
                if (i >> axis) & 1:
                    new_offset[axis] += 1
            collect_nodes_with_morton(child, node_list, depth + 1, tuple(new_offset))

def interleave_bits(x, y, z):
    """
    Standard bit interleaving for Morton code [G].
    """
    # Production implementations use lookup tables or bitwise operations.
    # This is a simplified representation for documentation.
    return (x | (y << 1) | (z << 2))
```
## Subtopics

### [Bottom-up Octree Construction](1590-bottom-up-octree-construction.md)
A method for compressing a 3D scalar array into an adaptive-resolution octree by recursively grouping 8 voxels into a parent node if they are identical (lossless) or have combined variance below a threshold (lossy), storing the average value at the parent. It solves the problem of efficiently storing and rendering large scalar fields by reducing memory footprint while enabling accelerated ray traversal via a simultaneously built min/max tree. It differs from top-down approaches by starting from the finest grid and merging up, and from static structures by integrating compression and acceleration data (min/max) into a single cohesive build process.

### [Packed Tensor Representation](1611-packed-tensor-representation.md)
Packed Tensor Representation is a data structure designed to store samples from rays that have varying numbers of valid samples, avoiding the memory inefficiency of fixed-size tensors with masks when large portions of space are empty. It packs per-sample ray indices into per-ray chunk information (start index and count) for flattened tensors. This approach is similar to those used in Instant-NGP [17] and PyTorch3D [22], but is explicitly described here as a solution to the memory inefficiency of storing data as a tensor with shape $(n\_rays, n\_samples, ...)$ and an extra mask.

### [Octree Volume Structure](1978-octree-volume-structure.md)
The Octree Volume Structure is a memory-efficient storage scheme for 3D scalar fields that represents an octree by storing nodes corresponding to the parents of logical octree nodes. It solves the problem of high memory overhead in standard octree representations by eliminating explicit leaf nodes for the deepest level of interior nodes, instead embedding scalar values directly within parent structures. This approach serves as a lossless compression format for scalar data while simultaneously providing an acceleration structure for ray traversal via a built-in min/max tree. It differs from standard octrees by using "cap" nodes at the maximum depth and "scalar leaves" embedded in interior nodes, rather than distinct leaf node types.

### [Octree Volume Compression](1979-octree-volume-compression.md)
Octree Volume Compression is a lossless (or semi-lossless) technique that reduces the memory footprint of structured volume data by converting it into an octree structure and consolidating voxels with zero variance. It achieves compression factors of 3 to 5 for typical simulation data by exploiting spatial coherence, where large uniform regions are represented by single nodes. It differs from sibling methods like [SVO Node Encoding](2705-svo-node-encoding.md) by focusing on the initial construction-phase compression of the raw grid into the hierarchy, often utilizing iso-range clamping to discard irrelevant data values before or during tree construction.

### [Sparse Voxel Octree Attribute Encoding (Neighbor)](2641-sparse-voxel-octree-attribute-encoding-neighbor.md)
Encode the attribute of a sparse voxel octree into deltas from its neighbors.

### [SVO Node Encoding](2705-svo-node-encoding.md)
A 32-bit integer encoding used for octree nodes. The most significant bit (MSB) distinguishes between leaf nodes and internal nodes (addresses).

### [Sparse Voxel Octree Structure Encoding/Decoding](3012-sparse-voxel-octree-structure-encoding-decoding.md)
CPU functions to encode and decode the sparse voxel octree structure itself.

### [Hierarchical Data Structure Storage](3333-hierarchical-data-structure-storage.md)

## Operating characteristics
- **Time Complexity:** Construction is $O(N)$ for BFS or $O(N \log N)$ for Morton sorting, where $N$ is the number of nodes [G]. Traversal complexity is Not given in the sources, but the method is explicitly noted as unsuitable for efficient traversal [S1].
- **Memory Complexity:** $O(N)$, where $N$ is the number of non-empty nodes. It eliminates pointer overhead, reducing memory usage compared to pointer-based octrees [S1].
- **Measured Performance:** Schnabel and Klein [S1] report significant memory reduction for point clouds discretized into octrees, suitable for streaming.
- **Strengths:** Compact storage, no pointer overhead, suitable for serialization and streaming [S1].
- **Limitations:** Not suitable for efficient traversal [S1].
- **When to use:** Persistent storage, streaming, bandwidth-constrained environments [S1].
- **When not to use:** Real-time rendering requiring fast traversal [S1].
## Background
Implicit Octree Storage was proposed by Schnabel and Klein [S1] to decrease the memory consumption of point clouds discretized into octrees. Their work focused on 4K^3 resolution octrees and demonstrated that sorting the tree in an implicit order (e.g., breadth-first) allows storage without pointers, significantly reducing size for streaming or storage [S1]. This approach is related to subtree merging techniques by Webber and Dillencourt [S1], which compress quadtrees by merging common subtrees.
## Sources
[S1] High resolution sparse voxel DAGs, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
