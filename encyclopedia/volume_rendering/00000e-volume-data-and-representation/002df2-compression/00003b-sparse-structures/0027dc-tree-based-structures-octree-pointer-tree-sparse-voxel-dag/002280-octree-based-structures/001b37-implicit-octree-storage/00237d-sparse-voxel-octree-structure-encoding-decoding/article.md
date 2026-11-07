# Sparse Voxel Octree Structure Encoding/Decoding


## Overview
Sparse Voxel Octree (SVO) Structure Encoding/Decoding refers to the CPU-side algorithms responsible for serializing and deserializing the hierarchical topology of a sparse voxel octree [S1, unofficial]. It solves the problem of converting an in-memory tree data structure into a compact, persistent byte stream (and vice versa) for storage or network transmission. Unlike sibling methods that focus on attribute compression or file system indexing, this method specifically handles the structural encoding of node pointers and empty space using Morton codes [S1, S2, unofficial]. It builds on [Implicit Octree Storage](939-implicit-octree-storage.md) by providing the concrete serialization logic for the implicit structure.

## How it works
### Inputs and Outputs
- **Input**: An in-memory Sparse Voxel Octree structure containing nodes with children pointers and Morton codes representing spatial position.
- **Output**: A contiguous byte buffer (file) containing the encoded octree structure, including headers, root nodes, and compressed data if enabled.

### Data Structures
The core data structure is the Sparse Voxel Octree node. While exact internal layouts vary by implementation, the encoding process relies on:
1.  **Morton Code**: A 64-bit integer representing the 3D position of a voxel via Z-order curve mapping.
2.  **Node Structure**: Typically contains a bitmask or array of 8 children pointers. In implicit storage, empty nodes may be represented by null pointers or specific flags.
3.  **Header**: Metadata including compression type, grid size, node count, and data position offsets [S1, S2, unofficial].

### Steps
The encoding and decoding processes are inverse operations. The following steps describe the encoding pipeline, which includes finalization and optional compression [S2, unofficial]:

1.  **Traverse and Encode**: Traverse the octree in Morton order. For each node, determine its children status.
    *   If a node has children, encode the children's Morton codes and recursively encode their subtrees.
    *   If a node is a leaf or empty, encode the appropriate flag or value.
    *   *Note*: The specific bit-packing of the node structure (e.g., run-length encoding of empty children) is implementation-specific but generally aims to minimize storage for sparse data [S2, unofficial].
2.  **Finalization (Padding)**: Ensure the tree is complete up to the maximum desired depth.
    *   If the current maximum Morton depth is less than the maximum Morton depth, call `FastAddEmptyVoxel` with a difference of +1 to pad empty voxels [S2, unofficial].
3.  **Write Root**: Write the root node (typically `m_Children[0][0]` or the root of the Morton-ordered list) to the output stream [S2, unofficial].
4.  **Write Header**: Write the file header containing metadata (compression type, grid size, node count, data positions) [S1, S2, unofficial].
5.  **Compress (Optional)**: If the compression mode is not `NONE`, apply incremental compression to the file data [S2, unofficial].
6.  **Close**: Finalize the file handle [S2, unofficial].

Decoding reverses this: read the header, decompress if necessary, parse the root, and recursively reconstruct the tree by interpreting Morton codes and child pointers.

### Formulas
No specific mathematical formulas are defined for the structural encoding itself in the sources, as it is primarily a data serialization process. However, the spatial mapping relies on the Morton code, which maps 3D coordinates $(x, y, z)$ to a 1D integer $m$:
$$ m = \text{interleave}(x, y, z) $$
Where $\text{interleave}$ is the bitwise Z-order curve function [S2, unofficial].

### Parameters
- **Compression Mode**: Can be `NONE` or a specific compression algorithm (e.g., LZ4, Zstd). Default is often `NONE` for raw structural access [S2, unofficial].
- **Max Morton Depth**: The maximum depth of the octree, determining the finest resolution. Not given in the sources.
- **Grid Size**: The dimensions of the bounding box in voxels. Not given in the sources.

### Edge Cases and Pitfalls
- **Precision**: Morton codes must be handled with care to avoid overflow at high depths. 64-bit integers are typically required for large volumes [S2, unofficial].
- **Boundaries**: The encoding must correctly handle the bounding box (AABB) padding to ensure all voxels align with the grid [S1, unofficial].
- **Empty Voxels**: Efficiently encoding large empty regions is critical. Failure to pad or compress empty nodes leads to bloated file sizes [S2, unofficial].
- **Incomplete Trees**: The finalization step is crucial to ensure the tree is valid for rendering, especially if the input data has gaps at the maximum depth [S2, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_sparse_voxel_octree(root_node, max_depth, compression_mode, output_file):
    """
    Encodes a Sparse Voxel Octree into a file.
    
    Args:
        root_node: The root node of the octree.
        max_depth: The maximum depth of the octree.
        compression_mode: Compression type (e.g., 'NONE', 'LZ4').
        output_file: The file path or buffer to write to.
    """
    # Step 1: Finalize the octree by padding empty voxels
    current_max_depth = get_max_morton_depth(root_node)
    if current_max_depth < max_depth:
        # Pad empty voxels to match max Morton depth
        fast_add_empty_voxel(root_node, difference=1)
    
    # Step 2: Write root node
    # The root node is typically the first element in the Morton-ordered list
    write_node(output_file, root_node.children[0][0])
    
    # Step 3: Write header
    # Header contains compression type, grid size, node count, data position
    write_header(output_file, {
        'compression': compression_mode,
        'grid_size': get_grid_size(root_node),
        'node_count': count_nodes(root_node),
        'data_position': get_data_offset(output_file)
    })
    
    # Step 4: Compress if necessary
    if compression_mode != 'NONE':
        compress_file_incremental(output_file, True)
    
    # Step 5: Close file
    close_file(output_file)

def decode_sparse_voxel_octree(input_file):
    """
    Decodes a Sparse Voxel Octree from a file.
    
    Args:
        input_file: The file path or buffer to read from.
        
    Returns:
        The root node of the reconstructed octree.
    """
    # Step 1: Read header
    header = read_header(input_file)
    
    # Step 2: Decompress if necessary
    if header['compression'] != 'NONE':
        decompress_file_incremental(input_file)
    
    # Step 3: Read root node
    root_node = read_node(input_file)
    
    # Step 4: Recursively reconstruct the tree
    # This step is implementation-specific but involves reading children
    # based on Morton codes and node flags.
    reconstruct_tree(root_node, input_file)
    
    return root_node
```

## Subtopics

### [Octree File I/O Format](5032-octree-file-i-o-format.md)
This format stores octree metadata in a separate text header file containing compression type, grid size, node count, and data position. It solves the problem of persistent storage for CPU-based ray casting by separating structural metadata from voxel data, allowing the renderer to map memory efficiently without parsing complex binary headers for every access. It differs from file-system-based indexing methods (like [Octree-based Space Subdivision with File System Indexing](5866-octree-based-space-subdivision-with-file-system-indexing.md)) by storing the entire octree structure in a single contiguous file rather than distributing nodes across the directory hierarchy.

### [Tolerance-based Voxel Fusion](5866-tolerance-based-voxel-fusion.md)
Tolerance-based Voxel Fusion is a performance optimization strategy that renders volume data at variable resolutions across the spatial domain by fusing voxels with similar values. It solves the trade-off between rendering fidelity and computational cost by coarsening the representation in regions where visual detail is less critical, achieving significant savings in storage and rendering time without perceptible quality loss in moderate coarsening modes. It differs from image-space skipping, which creates blocky artifacts, by operating in the spatial domain to maintain visual continuity.

### [Temporal Difference Encoding](5869-temporal-difference-encoding.md)
Temporal Difference Encoding is a compression technique for time-varying volume data that exploits high temporal correlation between consecutive time steps. It predicts each sample based on past data and encodes the difference between the prediction and the actual value. Structurally, it merges octrees from consecutive time steps by storing pointers to unchanged subtrees from previous steps instead of duplicating data. Unlike methods that treat all dimensions uniformly, this approach applies difference encoding specifically to the time domain.

### [Sparse Voxel DAG](5984-sparse-voxel-dag.md)
Sparse Voxel DAG is a compression technique for sparse voxel octrees that reduces memory footprint by merging identical nodes and redirecting parent pointers to single shared instances. It builds on [Sparse Voxel Octree Structure Encoding/Decoding](3012-sparse-voxel-octree-structure-encoding-decoding.md) by introducing node deduplication, distinguishing itself from standard SVOs which store separate nodes for identical subtrees.

### Octree Finalization & Empty Voxel Padding
Finalizes the octree by padding empty voxels to match maximum Morton code depth, writing the root node, header, and optionally compressing [S2, unofficial]. The steps are: 1. If current Morton depth < max Morton depth, call `FastAddEmptyVoxel` with difference + 1. 2. Write root node (`m_Children[0][0]`). 3. Write header. 4. If compression != `NONE`, call `CompressFileIncremental(true)`. 5. Close file [S2, unofficial].

### Sparse Voxel Octree Attribute Encoding (Light)
A storage method for light values (0-15) that packs four 4-bit nibbles into a single 32-bit integer per 2x2x2 voxel region, reducing memory overhead and removing the need for the lowest octree layer [S3, unofficial]. The steps are: 1. Initialize light values to 0 for non-illuminated blocks. 2. Assign 1-15 to light-emitting blocks. 3. Propagate light by decrementing the value by 1 for each neighboring block until it reaches 0. 4. Pack four 4-bit light values (one per voxel in a 2x2x2 region) into a single 32-bit integer. 5. Store packed integers in a 4-layer octree per 16x16x16 area. 6. Stack multiple octrees vertically to cover the full 256-block chunk height [S3, unofficial]. The formula for light propagation is:
$$ \text{light}(Pos) = \max(\{0\} \cup \{LVal - \|LPos - Pos\|_1 \mid (LPos, LVal) \in \text{Lights}\}) $$
The layout stores each 2x2x2 voxel region as a single 32-bit integer, with four 4-bit nibbles representing the light value of each voxel. The octree uses 4 layers instead of 5 due to the 2x2x2 packing granularity. Multiple octrees are stacked vertically to cover the 256-block chunk height [S3, unofficial]. Unlike naive byte-per-block storage which wastes 50% of allocated memory per block, bit-packing utilizes 100% of the allocated 32-bit integer space for four voxels [S3, unofficial].

## Operating characteristics
- **Time Complexity**: Encoding is $O(N)$ where $N$ is the number of nodes in the octree, assuming constant-time Morton code generation and file I/O. Decoding is also $O(N)$.
- **Memory Complexity**: The encoded file size is proportional to the number of non-empty nodes and the compression ratio. In-memory, the structure requires $O(N)$ space for the tree nodes.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales well with sparse data, as empty regions are not explicitly stored. Compression can further reduce size for dense data.
- **Strengths**: Efficient storage for sparse volumes; supports incremental updates and compression; separates metadata from data for efficient access.
- **Limitations**: Requires CPU-side processing for encoding/decoding; Morton code generation can be computationally expensive for very large volumes; compression adds overhead.
- **When to use**: When persistent storage or network transmission of sparse voxel octrees is required, especially for CPU-based rendering.
- **When not to use**: When real-time GPU-only rendering is required without CPU preprocessing; when data is extremely dense and compression offers little benefit.

## Background
The concept of Sparse Voxel Octrees was introduced by Kniss et al. in 2002 [S1, S2, unofficial]. The specific CPU encoding/decoding functions and file formats described in the sources are from unofficial implementations and theses, such as `o-voxel-gpu` [S1, unofficial] and a Master's Thesis by Giles Penfold [S2, unofficial]. The light encoding method is from a sandbox voxel game engine implementation [S3, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] GilesPenfold/MasterThesisSVO, https://github.com/GilesPenfold/MasterThesisSVO, fetched 2026-10-26 (unofficial)
[S3] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
