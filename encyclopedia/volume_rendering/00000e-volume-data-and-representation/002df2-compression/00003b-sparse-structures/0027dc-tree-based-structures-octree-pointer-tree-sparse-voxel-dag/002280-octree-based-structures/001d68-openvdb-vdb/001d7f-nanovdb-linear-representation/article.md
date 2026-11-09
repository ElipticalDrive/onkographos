# NanoVDB Linear Representation

Also called: Linear VDB, NanoVDB, NanoVDB Museth, Open-VDB GPU Port, VDB / NanoVDB

## Overview
NanoVDB is a linear, GPU-optimized implementation of the OpenVDB sparse voxel structure that eliminates CPU-dependent memory pointers, enabling compact memory copying via `memcpy` and efficient voxel retrieval on the GPU [S1][S2, unofficial]. It solves the problem of transferring and sampling large, sparse volumetric datasets on graphics hardware by serializing the tree topology and data into a single contiguous memory block, supporting 0th to 3rd order interpolation and gradient computation [S1]. Unlike its parent OpenVDB, which supports dynamic modification, NanoVDB is immutable and optimized strictly for retrieval, offering a 4–6× memory footprint reduction through variable bit-rate quantization [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** An OpenVDB grid (or equivalent sparse voxel structure) containing active voxels and tree topology.
*   **Output:** A linear, 32-byte aligned memory buffer containing the serialized NanoVDB structure, ready for GPU transfer [S1].

**Data Structures and Layout:**
The NanoVDB structure is a shallow B+tree serialized into a single continuous block of memory [S2, unofficial][S3].
*   **Alignment:** The entire structure is 32-byte aligned [S1].
*   **Tree Topology:** Static. The tree consists of an unbounded root, internal nodes of size $32^3$ and $16^3$, and leaf nodes of size $8^3$ [S3].
*   **Node Entries:** Each node entry is either a pointer to a child node or a constant "tile" value [S3].
*   **Memory Layout:** Unlike OpenVDB, NanoVDB does not support trailing optional blind data or separate memory blocks; it is a strict serialized layout [S4].

**Key Encoding (Build Algorithm):**
The conversion process involves constructing 64-bit keys from voxel coordinates $(x, y, z)$ to map sparse voxels into the linear structure [S4].

1.  **Step 2 (Coordinate Encoding):** A 64-bit key is constructed from signed coordinates:
    *   Lower 21 bits (blue): Signed $x$ coordinate, right-shifted by 12 bits.
    *   Next 21 bits (purple): Signed $y$ coordinate, right-shifted by 12 bits.
    *   Upper 21 bits (green): Signed $z$ coordinate, right-shifted by 12 bits.
    *   *Note:* The remaining bits are reserved or zeroed in this intermediate step [S4].

2.  **Step 5 (Final Key Construction):** A unique 64-bit key is constructed for the final hash table entry:
    *   Lower 9 bits (gray): Offset local into leaf nodes ($2^9 = 512 = 8^3$).
    *   Next 12 bits (blue): Local offsets into lower nodes ($2^{12} = 4096 = 16^3$).
    *   Next 15 bits (purple): Local offsets into upper nodes ($2^{15} = 32768 = 32^3$).
    *   Upper 28 bits (green): Tile ID into the hash table of the root node.
    *   *Constraint:* The 28-bit tile ID limits the root hash table to $2^{28} = 268,435,456$ entries [S4].

**Conversion and Serialization Steps:**
1.  **Conversion:** Convert the source OpenVDB grid to NanoVDB using provided builtin tools. The resulting structure is linear in memory and immutable [S1].
2.  **Quantization:** Apply variable bit-rate quantization to leaf values to compress the volume data [S2, unofficial].
3.  **Dithering:** Apply dithering to the quantized values to randomize quantization noise, reducing visual artifacts [S2, unofficial].
4.  **Serialization:** Serialize the entire tree (topology and quantized data) into a single continuous block of memory [S2, unofficial].
5.  **Transfer:** Copy the linear buffer to GPU memory using `cudaMemcpy()` [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def convert_openvdb_to_nanovdb(openvdb_grid):
    """
    Converts an OpenVDB grid to a linear NanoVDB memory buffer.
    """
    # 1. Validate input
    if not is_valid_openvdb(openvdb_grid):
        raise ValueError("Invalid OpenVDB grid")

    # 2. Serialize Tree Topology
    # Flatten the shallow B+tree (root -> 32^3 -> 16^3 -> 8^3 leaves)
    # into a linear array of nodes.
    linear_nodes = serialize_tree(openvdb_grid.root)

    # 3. Quantize Leaf Values
    # Apply variable bit-rate quantization to active voxels in leaf nodes
    quantized_leaves = quantize_values(linear_nodes.leaves, bits_per_value=8) # Typical value [S2, unofficial]

    # 4. Dither Quantization Noise
    # Randomize noise to prevent banding artifacts
    dithered_leaves = apply_dithering(quantized_leaves)

    # 5. Construct Linear Memory Buffer
    # Allocate 32-byte aligned memory
    buffer_size = calculate_buffer_size(linear_nodes, dithered_leaves)
    buffer = allocate_aligned_memory(buffer_size, alignment=32)

    # Copy topology and data into buffer
    copy_topology_to_buffer(linear_nodes, buffer)
    copy_quantized_data_to_buffer(dithered_leaves, buffer)

    return buffer

def serialize_tree(root_node):
    """
    Flattens the VDB tree into a linear structure.
    VDB is a shallow B+tree: unbounded root, internal nodes 32^3 and 16^3, leaves 8^3.
    """
    nodes = []
    
    # Process Root
    # Root entries are either child pointers or tile values
    for entry in root_node.entries:
        if entry.is_child_pointer:
            # Recursively process child nodes (32^3 or 16^3)
            child_nodes = serialize_tree(entry.child)
            nodes.extend(child_nodes)
        else:
            # Store tile value directly
            nodes.append(TileNode(value=entry.tile_value))
            
    return nodes

def quantize_values(leaves, bits_per_value=8):
    """
    Applies variable bit-rate quantization to leaf voxel values.
    """
    quantized = []
    for leaf in leaves:
        for voxel in leaf.active_voxels:
            # Quantize float value to integer representation
            q_val = round(voxel.value * (2**bits_per_value - 1))
            quantized.append(q_val)
    return quantized

def apply_dithering(values):
    """
    Adds random noise to quantized values to randomize quantization error.
    """
    dithered = []
    for val in values:
        noise = random.uniform(-0.5, 0.5)
        dithered.append(val + noise)
    return dithered
```

**Parameters:**
*   **Alignment:** 32 bytes [S1].
*   **Quantization Bits:** Not explicitly fixed in sources, but described as "variable bit-rate" [S2, unofficial]. Typical implementations use 8-bit or lower for significant compression [S2, unofficial].
*   **Interpolation Order:** Supports 0th to 3rd order interpolation [S1].

**Edge Cases and Pitfalls:**
*   **Modification:** NanoVDB does not support arbitrary modification. It is optimized for retrieval/sampling only. Modification is rudimentary and limited to altering values of voxels that are already active in the VDB [S1].
*   **Static Topology:** The tree topology must be static. Dynamic tree modifications are not supported because the entire tree is serialized into a single continuous block of memory [S2, unofficial].
*   **Root Hash Table Limit:** The 28-bit tile ID imposes a limit of $2^{28}$ entries in the root hash table. While extremely unlikely to be exceeded (each entry spans $4096^3$ voxels), it is a hard constraint [S4].
*   **Precision:** Quantization introduces small errors. This is acceptable for rendering but may affect simulation accuracy [S2, unofficial].

## Subtopics

### [NanoVDB Volume Loading and GPU Transfer](4997-nanovdb-volume-loading-and-gpu-transfer.md)
This method describes the process of reading a NanoVDB volume file from disk, validating its data type, and transferring the raw voxel data to GPU memory for rendering. It solves the problem of bridging the gap between the CPU-side NanoVDB linear representation and the GPU-side compute buffers required by ray marching or splatting kernels. It is distinct from general VDB loading because it operates on the linear, contiguous memory layout defined in [NanoVDB Linear Representation](1527-nanovdb-linear-representation.md), allowing for direct `memcpy`-style transfers without complex tree traversal on the CPU.

### [Accelerated Sparse Convolutional Operators](9795-accelerated-sparse-convolutional-operators.md)
Accelerated Sparse Convolutional Operators are a set of GPU-optimized CUDA kernels designed to perform stencil-based convolutions on sparse voxel data stored in the NanoVDB format. The method solves the performance bottleneck of applying 3D convolutional filters to sparse grids by adapting the computational strategy to the local occupancy density and feature depth of the input data. It sets itself apart from generic sparse convolution libraries (like SpConv v2) by leveraging the inherent spatial locality of NanoVDB's linear indexing scheme and utilizing NVIDIA tensor cores for dense sub-blocks, thereby achieving higher compute bandwidth utilization on specific sparsity patterns.

### NanoVDB Memory Layout
A serialized memory layout for NanoVDB, unlike OpenVDB, which supports trailing optional blind data (e.g., ML features) or separate memory blocks [S4]. The layout is strictly linear and 32-byte aligned [S1]. The 64-bit key breakdown for voxel coordinates in build algorithm step 2 encodes signed x, y, z coordinates right-shifted 12 bits into 21-bit fields each [S4]. In step 5, the 64-bit key encodes local offsets into leaf ($2^9$), lower ($2^{12}$), and upper ($2^{15}$) nodes, plus a 28-bit tile ID into the root hash table [S4]. The 28-bit tile ID imposes a limit of $2^{28}$ entries in the root hash table, though this is extremely unlikely to be exceeded since each entry corresponds to a child node of the root spanning an index domain of size $4096^3$ voxels [S4].

## Operating characteristics
*   **Time Complexity:** Retrieval is $O(1)$ for active voxels due to the linear hash table lookup [S3]. Conversion from OpenVDB is linear with respect to the number of active voxels and tree nodes [S4].
*   **Memory Complexity:** Memory footprint is reduced by 4–6× compared to OpenVDB representations [S2, unofficial]. The linear layout allows for efficient `memcpy` transfers, reducing CPU-GPU transfer overhead [S1].
*   **Measured Performance:** Fast, low-overhead, and on-the-fly decompression during rendering [S1]. Works well in incoherent Monte Carlo path tracing contexts [S1].
*   **Strengths:**
    *   GPU-optimized with no CPU address space dependencies [S1].
    *   Compact memory layout enables efficient transfer and caching [S1].
    *   Supports high-order interpolation and gradient computation [S1].
    *   Compatible with multiple graphics APIs (CUDA, DX12, OptiX, OpenGL, OpenCL, Vulkan, GLSL) [S2, unofficial].
*   **Limitations:**
    *   Immutable; no dynamic tree modifications [S2, unofficial].
    *   Rudimentary modification support (only active voxels) [S1].
    *   Quantization errors may affect precision-sensitive applications [S2, unofficial].
*   **When to Use:** Ideal for rendering and simulation applications where the volume data is static and large [S2, unofficial]. Suitable for film and simulation volumes, not game edits [S3].
*   **When Not to Use:** Not suitable for dynamic scenes requiring frequent tree modifications or arbitrary voxel insertion/deletion [S1].

## Background
NanoVDB was introduced by Mattias Museth in 2021 as a linear, GPU-optimized implementation of OpenVDB [S1][S2, unofficial]. It addresses the limitations of OpenVDB, which relies on CPU-specific libraries and memory pointers, by providing a dependency-free C++/C99 implementation of the VDB tree structure [S2, unofficial]. The design is based on the shallow B+tree structure of VDB, which uses an unbounded root, internal nodes of $32^3$ and $16^3$, and $8^3$ leaves [S3]. NanoVDB is aimed at simulation and film volumes, porting the VDB structure to the GPU for ray tracing [S3]. It is used for on-the-fly decompression during rendering [S1].

## Sources
[S1] GPU Volume Rendering with Hierarchical Compression Using VDB, https://arxiv.org/pdf/2504.04564, fetched 2026-10-26
[S2] NeuralVDB: High-resolution Sparse Volume Representation using Hierarchical Neural Networks | NVIDIA, https://www.youtube.com/watch?v=IfgeduMv_ns, fetched 2026-11-04 (unofficial)
[S3] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
[S4] fVDB: A Deep-Learning Framework for Sparse, Large-Scale, and High-Performance Spatial Intelligence, https://research.nvidia.com/labs/prl/publication/williams2024fvdb/, fetched 2026-10-30
