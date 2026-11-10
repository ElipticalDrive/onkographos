# Sparse Voxel Octree Structure Encoding/Decoding


## Overview
Sparse Voxel Octree (SVO) Structure Encoding/Decoding refers to the CPU-side algorithms responsible for serializing and deserializing the hierarchical topology of a sparse voxel octree [S1, S2, unofficial]. It solves the problem of converting an in-memory tree structure into a compact, persistent byte stream (or vice versa) for storage or transmission, separating structural metadata from the actual voxel data [S1, S2, unofficial]. Unlike sibling methods that focus on attribute compression or file-system indexing, this method specifically handles the Morton-code-based node ordering and empty-voxel padding required to reconstruct the exact spatial hierarchy on the CPU [S1, S2, unofficial].

## How it works
### Inputs and Outputs
*   **Input:** An in-memory Sparse Voxel Octree data structure, typically defined by a root node containing child pointers and a maximum depth (Morton code depth).
*   **Output:** A contiguous byte buffer or file stream containing the encoded octree structure, including headers, node data, and padding.

### Data Structures
The structure relies on the implicit octree storage model where nodes are ordered by Morton codes [Parent].
*   **Node:** Contains an 8-bit child mask (indicating which of the 8 children exist) and pointers/references to child nodes.
*   **Header:** Stores metadata such as compression type, grid size, node count, and data position offsets [S1, S2, unofficial].
*   **Morton Code:** A 3D coordinate mapped to a 1D integer to preserve spatial locality.

### Steps
1.  **Finalization and Padding:**
    *   Ensure the octree depth matches the maximum Morton code depth. If the current depth is less than the maximum, insert empty voxels to fill the hierarchy up to the max depth [S2, unofficial].
    *   This is achieved by calling `FastAddEmptyVoxel` with a difference of +1 for missing branches [S2, unofficial].
2.  **Sorting:**
    *   Sort all leaf nodes (and internal nodes if required by the specific encoding variant) by their Morton codes to ensure spatial locality in the output stream [S1, S2, unofficial].
3.  **Header Writing:**
    *   Write the header containing compression type, grid dimensions, total node count, and the starting offset of the voxel data [S1, S2, unofficial].
4.  **Node Encoding:**
    *   Write the root node explicitly [S2, unofficial].
    *   Iterate through the sorted nodes and write their child masks and child references. References are typically stored as offsets relative to the start of the node data or as absolute positions in the stream [S1, S2, unofficial].
5.  **Compression (Optional):**
    *   If compression is enabled (e.g., `NONE` is not selected), apply incremental compression to the encoded file/stream [S2, unofficial].
6.  **Closing:**
    *   Finalize the file or buffer [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_sparse_voxel_octree(root_node, max_depth, compression_type):
    """
    Encodes an in-memory SVO into a byte stream.
    """
    # 1. Finalization: Pad empty voxels to max depth
    if root_node.current_depth < max_depth:
        # Insert empty nodes to ensure full depth coverage
        fast_add_empty_voxel(root_node, depth_difference=1)
    
    # 2. Collect and Sort Nodes
    # Collect all nodes into a list
    nodes = collect_all_nodes(root_node)
    # Sort by Morton code to ensure spatial locality
    nodes.sort(key=lambda node: node.morton_code)
    
    # 3. Prepare Output Buffer
    buffer = bytearray()
    
    # 4. Write Header
    header = {
        "compression_type": compression_type,
        "grid_size": root_node.grid_size,
        "node_count": len(nodes),
        "data_position": len(buffer) # Placeholder, updated later or fixed size
    }
    write_header(buffer, header)
    
    # 5. Write Root Node
    # The root is written explicitly as per specification
    write_node(buffer, root_node)
    
    # 6. Write Remaining Nodes
    for node in nodes:
        if node is root_node:
            continue
        write_node(buffer, node)
        
    # 7. Compression
    if compression_type != CompressionType.NONE:
        compress_file_incremental(buffer, is_final=True)
        
    return buffer

def decode_sparse_voxel_octree(buffer):
    """
    Decodes a byte stream back into an in-memory SVO.
    """
    # 1. Read Header
    header = read_header(buffer)
    
    # 2. Decompression (if needed)
    if header.compression_type != CompressionType.NONE:
        decompress_file_incremental(buffer)
        
    # 3. Read Root Node
    root_node = read_node(buffer)
    
    # 4. Read Remaining Nodes and Reconstruct Tree
    # The order in the buffer corresponds to the Morton-sorted order
    # Child pointers are resolved using offsets or indices
    nodes = []
    for i in range(header.node_count - 1):
        node = read_node(buffer)
        nodes.append(node)
        
    # Re-link children based on offsets/indices stored in the nodes
    reconstruct_tree(root_node, nodes)
    
    return root_node
```

### Parameters
*   **max_depth:** The maximum depth of the octree, determined by the resolution of the voxel grid. Typical values depend on the scene scale and required detail (e.g., 10-20).
*   **compression_type:** Enum indicating whether to compress the structure (e.g., `NONE`, `RLE`, `LZ4`). Default is often `NONE` for CPU-based ray casting to avoid decompression overhead during traversal [S1, S2, unofficial].

### Edge Cases and Pitfalls
*   **Depth Mismatch:** If the octree is not padded to the maximum Morton code depth, decoding may fail or produce incorrect spatial mappings [S2, unofficial].
*   **Pointer Validity:** Offsets stored in child pointers must be valid relative to the start of the node data section. Incorrect offsets lead to corrupted tree reconstruction [S1, S2, unofficial].
*   **Empty Tree:** Handling an octree with no voxels requires writing a valid root node with an empty child mask [S2, unofficial].

## Subtopics

### [Octree File I/O Format](5032-octree-file-i-o-format.md)
This format stores octree metadata in a separate text header file containing compression type, grid size, node count, and data position. It solves the problem of persistent storage for CPU-based ray casting by separating structural metadata from voxel data, allowing the renderer to map memory efficiently without parsing complex binary headers for every access. It differs from file-system-based indexing methods (like [Octree-based Space Subdivision with File System Indexing](5866-octree-based-space-subdivision-with-file-system-indexing.md)) by storing the entire octree structure in a single contiguous file rather than distributing nodes across the directory hierarchy.

### [Tolerance-based Voxel Fusion](5866-tolerance-based-voxel-fusion.md)
Tolerance-based Voxel Fusion is a performance optimization strategy that renders volume data at variable resolutions across the spatial domain by fusing voxels with similar values. It solves the trade-off between rendering fidelity and computational cost by coarsening the representation in regions where visual detail is less critical, achieving significant savings in storage and rendering time without perceptible quality loss in moderate coarsening modes. It differs from image-space skipping, which creates blocky artifacts, by operating in the spatial domain to maintain visual continuity.

### [Temporal Difference Encoding](5869-temporal-difference-encoding.md)
Temporal Difference Encoding is a compression technique for time-varying volume data that exploits high temporal correlation between consecutive time steps. It predicts each sample based on past data and encodes the difference between the prediction and the actual value. Structurally, it merges octrees from consecutive time steps by storing pointers to unchanged subtrees from previous steps instead of duplicating data. Unlike methods that treat all dimensions uniformly, this approach applies difference encoding specifically to the time domain.

### [Sparse Voxel DAG](5984-sparse-voxel-dag.md)
Sparse Voxel DAG is a compression technique for sparse voxel octrees that reduces memory footprint by merging identical nodes and redirecting parent pointers to single shared instances. It builds on [Sparse Voxel Octree Structure Encoding/Decoding](3012-sparse-voxel-octree-structure-encoding-decoding.md) by introducing node deduplication, distinguishing itself from standard SVOs which store separate nodes for identical subtrees.

### [G-PCC PlenOctree Compression](11441-g-pcc-plenoctree-compression.md)
G-PCC PlenOctree Compression is a method for compressing PlenOctree models by encoding the geometry losslessly and the attributes (opacity $\sigma$ and spherical harmonic coefficients $k$) lossily using Geometry-based Point Cloud Coding (G-PCC) standards. It differs from sibling methods like [Octree File I/O Format](5032-octree-file-i-o-format.md) by treating the voxel grid as a point cloud to leverage standard PC codecs, and from [Sparse Voxel DAG](5984-sparse-voxel-dag.md) by preserving the explicit octree structure rather than merging identical nodes.

### Octree Finalization & Empty Voxel Padding
Finalizes the octree by padding empty voxels to match maximum Morton code depth, writing the root node, header, and optionally compressing [S2, unofficial].
1. If current Morton depth < max Morton depth, call `FastAddEmptyVoxel` with difference + 1 [S2, unofficial].
2. Write root node (`m_Children[0][0]`) [S2, unofficial].
3. Write header [S2, unofficial].
4. If compression != `NONE`, call `CompressFileIncremental(true)` [S2, unofficial].
5. Close file [S2, unofficial].

### Sparse Voxel Octree Attribute Encoding (Light)
A storage method for light values (0-15) that packs four 4-bit nibbles into a single 32-bit integer per 2x2x2 voxel region, reducing memory overhead and removing the need for the lowest octree layer [S3, unofficial].
1. Initialize light values to 0 for non-illuminated blocks [S3, unofficial].
2. Assign 1-15 to light-emitting blocks [S3, unofficial].
3. Propagate light by decrementing the value by 1 for each neighboring block until it reaches 0 [S3, unofficial].
4. Pack four 4-bit light values (one per voxel in a 2x2x2 region) into a single 32-bit integer [S3, unofficial].
5. Store packed integers in a 4-layer octree per 16x16x16 area [S3, unofficial].
6. Stack multiple octrees vertically to cover the full 256-block chunk height [S3, unofficial].
Formula: $light(Pos) = \max(\{0\} \cup \{LVal - \|LPos - Pos\|_1 \mid (LPos, LVal) \in Lights\})$ [S3, unofficial].
Layout: Each 2x2x2 voxel region is stored as a single 32-bit integer. Four 4-bit nibbles represent the light value of each voxel. The octree uses 4 layers instead of 5 due to the 2x2x2 packing granularity. Multiple octrees are stacked vertically to cover the 256-block chunk height [S3, unofficial].
Comparison: Unlike naive byte-per-block storage which wastes 50% of allocated memory per block, bit-packing utilizes 100% of the allocated 32-bit integer space for four voxels [S3, unofficial].

### Hyper Octree
A compact data structure for lossy compression of voxel data in an animation sequence, designed to reduce memory consumption [S4, unofficial].
1. Use a modified sparse voxel octree that exploits spatiotemporal coherency between each consecutive frame in a sequence [S4, unofficial].
2. Compress colours using the DXT1 format [S4, unofficial].
3. Sort or reorder colours using HSV values to place similar colours in close proximity in memory before DXT1 compression [S4, unofficial].
4. Reduce redundant data generated by the DXT1 format to further compress the data [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Encoding is dominated by the sorting step, $O(N \log N)$ where $N$ is the number of nodes, assuming Morton code calculation is $O(1)$ per node. Decoding is $O(N)$ as it involves sequential reading and pointer resolution.
*   **Memory Complexity:** The encoded size is proportional to the number of nodes $N$. Each node typically requires 1 byte for the child mask plus 4-8 bytes for child offsets/pointers, depending on the implementation [S1, S2, unofficial].
*   **Performance:** CPU-based decoding is fast due to sequential memory access patterns after sorting, making it suitable for real-time ray casting on the CPU [S1, S2, unofficial].
*   **Strengths:** Compact representation of sparse data; efficient spatial locality for ray traversal; simple header structure for quick metadata access [S1, S2, unofficial].
*   **Limitations:** Sorting overhead for very large trees; not inherently parallelizable for encoding; requires full tree reconstruction for random access if not indexed [S1, S2, unofficial].

## Background
The Sparse Voxel Octree (SVO) structure encoding/decoding methods described here originate from implementations in CPU-based volume rendering and voxel game engines. The specific CPU functions `encode_sparse_voxel_octree_cpu` and `decode_sparse_voxel_octree_cpu` are documented in the `o-voxel-gpu` repository [S1, unofficial] and a Master's thesis on SVOs [S2, unofficial]. The attribute encoding variants (Light, Hyper Octree) are derived from game engine architectures like DigBuild [S3, unofficial] and animation compression research [S4, unofficial]. These methods are used in real-time rendering applications where memory bandwidth and cache efficiency are critical, particularly for CPU ray casting [S1, S2, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] GilesPenfold/MasterThesisSVO, https://github.com/GilesPenfold/MasterThesisSVO, fetched 2026-10-26 (unofficial)
[S3] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
[S4] Compression of Sequential Voxel Data for Sequence-based Animation, https://core.ac.uk/download/620674356.pdf, fetched 2026-10-26 (unofficial)
