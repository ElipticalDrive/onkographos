# Sparse Voxel Octree Structure Encoding/Decoding


## Overview
This method defines the CPU-side algorithms for serializing and deserializing the structural topology of a Sparse Voxel Octree (SVO) [S1, unofficial]. It solves the problem of mapping a hierarchical spatial subdivision into a linear byte stream for storage or transmission, separating structural metadata from voxel attribute data [S1, unofficial]. It differs from file-system-based indexing methods by producing a single contiguous binary payload rather than distributing nodes across directory hierarchies [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** A populated Sparse Voxel Octree data structure containing nodes with children pointers and Morton codes; a target file handle or memory buffer.
**Outputs:** A serialized byte stream representing the octree structure.

**Data Structures:**
The SVO node structure typically contains:
- `m_Children`: An array of 8 pointers or indices to child nodes.
- `m_MortonCode`: The spatial key determining the node's position in the hierarchy.
- `m_Depth`: The current depth in the octree.

**Steps:**
1. **Finalization:** If the current maximum Morton code depth is less than the defined maximum depth, pad the tree with empty voxels to ensure a complete structure [S2, unofficial].
   - Call `FastAddEmptyVoxel` with a difference of +1 to extend the depth [S2, unofficial].
2. **Root Serialization:** Write the root node (`m_Children[0][0]`) to the output stream [S2, unofficial].
3. **Header Writing:** Write the file header containing metadata such as version, bounding box, and compression flags [S2, unofficial].
4. **Compression (Optional):** If the compression mode is not `NONE`, invoke the incremental compression routine [S2, unofficial].
   - Call `CompressFileIncremental(true)` [S2, unofficial].
5. **Closure:** Close the file handle or finalize the memory buffer [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_sparse_voxel_octree_cpu(svo, file_handle, compression_mode):
    """
    Encodes the SVO structure to a file.
    :param svo: The Sparse Voxel Octree object.
    :param file_handle: Open file object for writing.
    :param compression_mode: Enum indicating compression type (e.g., NONE, LZ4).
    """
    # Step 1: Finalize octree by padding empty voxels
    if svo.current_morton_depth < svo.max_morton_depth:
        # Extend depth to match maximum Morton code depth
        svo.fast_add_empty_voxel(difference=1)
    
    # Step 2: Write root node
    # Assuming root is accessible at index 0 of children array
    root_node = svo.root_node.m_children[0][0]
    write_node_to_stream(root_node, file_handle)
    
    # Step 3: Write header
    write_header(svo.metadata, file_handle)
    
    # Step 4: Compress if required
    if compression_mode != CompressionMode.NONE:
        # Incremental compression of the written data
        compress_file_incremental(file_handle, is_final=True)
    
    # Step 5: Close file
    file_handle.close()

def decode_sparse_voxel_octree_cpu(file_handle):
    """
    Decodes the SVO structure from a file.
    :param file_handle: Open file object for reading.
    :return: Reconstructed Sparse Voxel Octree object.
    """
    # Step 1: Read header
    metadata = read_header(file_handle)
    
    # Step 2: Decompress if necessary
    if metadata.compression_mode != CompressionMode.NONE:
        decompress_file_incremental(file_handle)
    
    # Step 3: Read root node
    root_node = read_node_from_stream(file_handle)
    
    # Step 4: Reconstruct tree recursively
    svo = SparseVoxelOctree()
    svo.root_node = reconstruct_tree(root_node, metadata.max_depth)
    
    return svo
```

**Parameters:**
- `compression_mode`: Typically `NONE` for raw storage or `LZ4`/`ZSTD` for compressed storage [S2, unofficial].
- `max_morton_depth`: The maximum depth of the octree, determining the finest voxel resolution [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Depth Mismatch:** If the current Morton depth is less than the maximum, empty voxels must be explicitly added to maintain structural integrity [S2, unofficial].
- **Precision:** Morton codes must be handled with care to avoid overflow at high depths [G].

## Subtopics

### [Octree File I/O Format](5032-octree-file-i-o-format.md)
This format defines the ASCII header and binary payload structure for serializing a Sparse Voxel Octree (SVO) to disk, enabling efficient loading and unloading of volumetric data. It solves the problem of persistent storage for CPU-based ray casting by separating structural metadata from voxel data, allowing the renderer to map memory efficiently without parsing complex binary headers for every access. It differs from file-system-based indexing methods (like [Octree-based Space Subdivision with File System Indexing](5866-octree-based-space-subdivision-with-file-system-indexing.md)) by storing the entire octree structure in a single contiguous file rather than distributing nodes across the directory hierarchy.

### [Tolerance-based Voxel Fusion](5866-tolerance-based-voxel-fusion.md)
Tolerance-based Voxel Fusion is a performance optimization strategy that renders volume data at variable resolutions across the spatial domain by fusing voxels with similar values. It solves the trade-off between rendering fidelity and computational cost by coarsening the representation in regions where visual detail is less critical, achieving significant savings in storage and rendering time without perceptible quality loss in moderate coarsening modes. It differs from image-space skipping, which creates blocky artifacts, by operating in the spatial domain to maintain visual continuity.

### [Temporal Difference Encoding](5869-temporal-difference-encoding.md)
Temporal Difference Encoding is a compression technique for time-varying volume data that exploits high temporal correlation between consecutive time steps. It predicts each sample based on past data and encodes the difference between the prediction and the actual value. Structurally, it merges octrees from consecutive time steps by storing pointers to unchanged subtrees from previous steps instead of duplicating data. Unlike methods that treat all dimensions uniformly, this approach applies difference encoding specifically to the time domain.

### [Sparse Voxel DAG](5984-sparse-voxel-dag.md)
Sparse Voxel DAG is a compression technique for sparse voxel octrees that reduces memory footprint by merging identical nodes and redirecting parent pointers to single shared instances. It builds on [Sparse Voxel Octree Structure Encoding/Decoding](3012-sparse-voxel-octree-structure-encoding-decoding.md) by introducing node deduplication, distinguishing itself from standard SVOs which store separate nodes for identical subtrees.

### Octree Finalization & Empty Voxel Padding
Finalizes the octree by padding empty voxels to match maximum Morton code depth, writing the root node, header, and optionally compressing [S2, unofficial].
1. If current Morton depth < max Morton depth, call `FastAddEmptyVoxel` with difference + 1 [S2, unofficial].
2. Write root node (`m_Children[0][0]`) [S2, unofficial].
3. Write header [S2, unofficial].
4. If compression != NONE, call `CompressFileIncremental(true)` [S2, unofficial].
5. Close file [S2, unofficial].

### Sparse Voxel Octree Attribute Encoding (Light)
A storage method for light values (0-15) that packs four 4-bit nibbles into a single 32-bit integer per 2x2x2 voxel region, reducing memory overhead and removing the need for the lowest octree layer [S3, unofficial].
1. Initialize light values to 0 for non-illuminated blocks [S3, unofficial].
2. Assign 1-15 to light-emitting blocks [S3, unofficial].
3. Propagate light by decrementing the value by 1 for each neighboring block until it reaches 0 [S3, unofficial].
4. Pack four 4-bit light values (one per voxel in a 2x2x2 region) into a single 32-bit integer [S3, unofficial].
5. Store packed integers in a 4-layer octree per 16x16x16 area [S3, unofficial].
6. Stack multiple octrees vertically to cover the full 256-block chunk height [S3, unofficial].

Formula:
$$ \text{light}(Pos) = \max(\{0\} \cup \{LVal - \|LPos - Pos\|_1 \mid (LPos, LVal) \in \text{Lights}\}) $$

Layout: Each 2x2x2 voxel region is stored as a single 32-bit integer. Four 4-bit nibbles represent the light value of each voxel. The octree uses 4 layers instead of 5 due to the 2x2x2 packing granularity. Multiple octrees are stacked vertically to cover the 256-block chunk height [S3, unofficial].

Comparison: Unlike naive byte-per-block storage which wastes 50% of allocated memory per block, bit-packing utilizes 100% of the allocated 32-bit integer space for four voxels [S3, unofficial].

## Operating characteristics
**Time Complexity:** Encoding is linear with respect to the number of nodes in the octree, $O(N)$, where $N$ is the number of non-empty nodes [S1, unofficial]. Decoding is similarly $O(N)$ [S1, unofficial].
**Memory Complexity:** The serialized size is proportional to the number of nodes plus header overhead [S2, unofficial].
**Performance:** CPU-based encoding/decoding is efficient for static scenes but may become a bottleneck for dynamic scenes if not parallelized [S1, unofficial].
**Strengths:** Simple implementation, deterministic output, easy to debug [S1, unofficial].
**Limitations:** Does not compress attribute data (color, density) by default; requires full tree traversal for serialization [S2, unofficial].

## Background
Originated in the context of CPU-based ray casting for volumetric data, specifically within the o-voxel-gpu project and related master's theses on SVOs [S1, unofficial][S2, unofficial]. It is used in real-time rendering engines where efficient loading of sparse volumetric data is required [S1, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] GilesPenfold/MasterThesisSVO, https://github.com/GilesPenfold/MasterThesisSVO, fetched 2026-10-26 (unofficial)
[S3] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
