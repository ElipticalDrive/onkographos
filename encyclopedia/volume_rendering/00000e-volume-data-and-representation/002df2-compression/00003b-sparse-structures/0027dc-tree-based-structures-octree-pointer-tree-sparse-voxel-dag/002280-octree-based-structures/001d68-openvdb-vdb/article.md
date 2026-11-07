# OpenVDB (VDB)

Also called: OpenVDB Hierarchical Compression, VDB

## Overview
OpenVDB is a sparse voxel representation designed for the hierarchical encoding and compression of volumetric grids, providing fast $O(1)$ random access for both modification and retrieval operations [S1]. It solves the problem of efficiently storing and accessing large, sparse volumetric datasets that exceed standard memory capacities by using a shallow, fixed-depth tree structure [S1]. Unlike general-purpose sparse structures, OpenVDB is optimized for production rendering and visual effects workflows, integrating directly into industry tools like Houdini and renderers like Cycles [S1].

## How it works
**Inputs:** A volumetric grid defined over a 3D domain.
**Outputs:** A serialized tree structure consisting of root, internal (tile), and leaf nodes, each containing specific bit masks and data pointers.

**Data Structures:**
The structure is a shallow tree with exactly four levels [S1].
1.  **Root Level:** Contains a variable number of children.
2.  **Internal Levels (Tiles):** Fixed number of child nodes per parent.
3.  **Leaf Level:** Stores a fixed number of voxels. The default leaf size is $8 \times 8 \times 8$ voxels [S1].

Each node at every level stores a 64-bit direct access bit mask [S1]. The content of this mask depends on the node's level:
*   **Internal Nodes (Tiles):** The 64-bit mask encodes the child ID or other representative values for the entire tile [S1].
*   **Leaf Nodes:** The 64-bit mask encodes the leaf's origin in 3D space using $3 \times 20$ bits (60 bits total) [S1]. The remaining bits encode additional metadata, such as whether the voxels are compressed or quantized [S1]. Additionally, leaf nodes contain a separate bit mask indicating which voxels within the leaf are active, and a raw pointer to the voxel data itself [S1].

**Algorithm:**
1.  **Grid Construction:** The input volume is subdivided recursively. The root node covers the entire domain. Internal nodes subdivide into children until the leaf resolution ($8 \times 8 \times 8$) is reached.
2.  **Mask Generation:** For each node, generate the 64-bit mask.
    *   For internal nodes, populate the mask with child identifiers or representative values.
    *   For leaf nodes, populate the mask with the 3D origin coordinates ($x, y, z$) using 20 bits per axis. Set metadata bits for compression/quantization status.
3.  **Data Storage:** For leaf nodes, store the active voxel values in the memory block pointed to by the leaf's raw pointer. Use the leaf's active mask to determine which entries in the pointer block are valid.
4.  **Serialization:** Serialize the tree nodes. The structure allows for out-of-core streaming [S1].

**Parameters:**
*   Leaf node voxel count: Default is $8 \times 8 \times 8$ [S1].

**Edge Cases and Pitfalls:**
*   **Memory Visibility:** The data structure prevails in address spaces visible to the CPU, requiring explicit handling for GPU access (e.g., via NanoVDB) [S1].
*   **Pointer Validity:** The raw pointer to voxel data is CPU-side. Direct dereferencing on the GPU without translation or linearization will fail [S1].

## Subtopics

### [VDB Volume Rendering with OpenVDB/NanoVDB](804-vdb-volume-rendering-with-openvdb-nanovdb.md)
Enables GPU rendering by combining CPU-side compression with GPU-side decompression.

### [NanoVDB Linear Representation](1527-nanovdb-linear-representation.md)
A linear, GPU-optimized snapshot of OpenVDB that eliminates CPU pointers.

### [VDB 3-4-5 Tree Traversal](4996-vdb-3-4-5-tree-traversal.md)
A traversal method using a 3-4-5 tree topology stored in compressed texture atlases.

### [OpenVDB File Format Parsing](5173-openvdb-file-format-parsing.md)
Binary parsing of the VDB file format to reconstruct the in-memory tree.

### [Sparse Field Training](8750-sparse-field-training.md)
Training MLPs by restricting sampling to active VDB voxels.

## Operating characteristics
*   **Time Complexity:** $O(1)$ random access for modification and retrieval [S1].
*   **Memory Complexity:** Sparse; only active voxels and necessary tree nodes are stored. The shallow tree (4 levels) minimizes traversal overhead [S1].
*   **Strengths:** Fast random access, out-of-core streaming support, well-established in production pipelines (Houdini, Cycles) [S1].
*   **Limitations:** Data resides in CPU-visible address space, requiring additional steps (like NanoVDB) for efficient GPU rendering [S1].
*   **When to use:** When working with large, sparse volumetric data in visual effects or scientific visualization where random access and modification are required [S1].
*   **When not to use:** When direct, high-performance GPU rendering is needed without CPU-GPU data transfer overhead (use NanoVDB instead) [S1].

## Background
OpenVDB was introduced by Museth [Mus13] and is an open-source library resulting from his work. It is popular in production rendering and industry-strength libraries optimized for NVIDIA GPUs [S1]. It is well-suited for visual effects rendering of volumetric data from fluid simulations [S1]. The structure shares similarities with B+trees [S1]. It is integrated into production renderers like Cycles [Ble24] and VFX software such as Houdini [Sid25] and Cinema [S1].

## Sources
[S1] GPU Volume Rendering with Hierarchical Compression Using VDB, https://arxiv.org/pdf/2504.04564, fetched 2026-10-26
