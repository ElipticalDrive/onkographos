# OpenVDB (VDB)
Also called: OpenVDB Hierarchical Compression, VDB
## Overview
OpenVDB is a sparse voxel representation used for the hierarchical encoding and compression of volumetric grids, featuring fast $O(1)$ random access for modification and retrieval operations [S1, unofficial]. It solves the problem of efficiently storing and accessing large, sparse volumetric datasets by using a shallow tree structure that balances memory overhead with traversal speed [S1, unofficial]. It is distinguished from its siblings by its specific four-level topology and 64-bit direct access bit masks that encode topology and data pointers, making it a standard in production visual effects and scientific visualization [S1, unofficial].
## How it works
**Inputs and Outputs:**
Input is a dense or sparse volumetric grid [S1, unofficial]. Output is a serialized OpenVDB tree structure containing hierarchical node data and voxel values [S1, unofficial].

**Data Structures:**
The structure is a shallow tree with four levels: Root, Internal (Tile), and Leaf [S1, unofficial].
1.  **Bit Masks:** Each level stores a 64-bit direct access bit mask [S1, unofficial].
    *   **Internal Nodes (Tiles):** The bit mask encodes the child ID or representative values for the entire tile [S1, unofficial].
    *   **Leaf Nodes:** The bit mask encodes the leaf origin using $3 \times 20$ bits [S1, unofficial]. Other bits encode metadata (e.g., compression status, quantization) [S1, unofficial].
2.  **Leaf Node Payload:**
    *   A bit mask indicating which voxels are active [S1, unofficial].
    *   A raw pointer to the voxel data array [S1, unofficial].
3.  **Voxel Data:** Stored in arrays pointed to by leaf nodes [S1, unofficial].

**Parameters:**
*   **Leaf Node Size:** Default is $8 \times 8 \times 8$ voxels [S1, unofficial].
*   **Bit Mask Width:** Fixed at 64 bits per level [S1, unofficial].

**Algorithm (Tree Construction and Access):**
The tree is constructed to allow $O(1)$ access [S1, unofficial]. The root has a variable number of children, while internal nodes have a fixed number of children [S1, unofficial].

1.  **Initialization:** Create the root node [S1, unofficial].
2.  **Insertion/Modification:**
    *   Calculate the node index for the target coordinate [S1, unofficial].
    *   Use the 64-bit bit mask at the current level to determine the child node index [S1, unofficial].
    *   If the child does not exist, create it [S1, unofficial].
    *   Recurse to the next level until the Leaf level is reached [S1, unofficial].
3.  **Leaf Access:**
    *   At the leaf level, use the 64-bit mask to decode the leaf origin ($3 \times 20$ bits) and metadata [S1, unofficial].
    *   Use the active voxel bit mask to locate the specific voxel within the leaf's data array [S1, unofficial].
    *   Return the value via the raw pointer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_voxel(root, coords):
    """
    Retrieve a voxel value from the OpenVDB tree.
    coords: tuple (x, y, z) representing the voxel position.
    """
    node = root
    
    # Traverse levels 0 to 2 (Root and Internal nodes)
    for level in range(3):
        # [G] Exact bit extraction from the 64-bit mask depends on OpenVDB version.
        # Standard approach: shift coordinate bits by level-specific offsets.
        child_index = (coords[0] >> (level * 3)) & 0x7
        
        if child_index is None or not node.has_child(child_index):
            return None # Voxel not present (empty space)
        
        node = node.children[child_index]
        
    # Now node is a Leaf
    # Decode leaf origin from the 64-bit mask (3 * 20 bits)
    # [G] Sources do not specify exact bit positions; standard extraction assumes contiguous blocks.
    leaf_origin_x = (node.bit_mask >> 40) & 0xFFFFF
    leaf_origin_y = (node.bit_mask >> 20) & 0xFFFFF
    leaf_origin_z = node.bit_mask & 0xFFFFF
    leaf_origin = (leaf_origin_x, leaf_origin_y, leaf_origin_z)
    
    # Calculate local coordinates within the leaf
    local_coords = (coords[0] - leaf_origin[0], 
                    coords[1] - leaf_origin[1], 
                    coords[2] - leaf_origin[2])
    
    # Check if the voxel is active using the leaf's active bit mask
    local_idx = local_coords[0] + local_coords[1] * 8 + local_coords[2] * 64
    if not (node.active_mask >> local_idx) & 1:
        return None # Voxel is inactive (e.g., zero or background)
    
    # Retrieve value from the raw data pointer
    data_index = local_idx
    return node.data_ptr[data_index]
```

**Edge Cases and Pitfalls:**
*   **Empty Space:** If a child node does not exist, the traversal stops, and the value is considered background/empty [S1, unofficial].
*   **Inactive Voxels:** Even if a leaf exists, individual voxels may be inactive. The active bit mask must be checked [S1, unofficial].
*   **Precision:** The leaf origin is encoded in 20 bits per axis. Ensure coordinates fit within this range [S1, unofficial].
*   **Memory:** The structure relies on raw pointers. The data prevails in address spaces visible to the CPU [S1, unofficial].

## Subtopics

### [VDB Volume Rendering with OpenVDB/NanoVDB](804-vdb-volume-rendering-with-openvdb-nanovdb.md)
This method enables GPU volume rendering of large, sparse volumetric datasets by combining offline fixed-rate lossy compression on the CPU (using OpenVDB) with fast, low-overhead on-the-fly decompression and sampling on the GPU (using NanoVDB). It solves the problem of rendering volumes that exceed single-GPU memory capacity by significantly reducing the memory footprint while maintaining random access patterns suitable for incoherent workloads like path tracing. Unlike hierarchical space-skipping approaches which show diminishing returns with RGBA transfer functions, this method typically employs a uniform majorant grid for traversal acceleration to facilitate direct comparison with dense texture rendering.

### [NanoVDB Linear Representation](1527-nanovdb-linear-representation.md)
A linear, GPU-optimized implementation of VDB that does not depend on CPU address spaces, allowing compact copying via memcpy and optimized voxel retrieval on the GPU.

### [VDB 3-4-5 Tree Traversal](4996-vdb-3-4-5-tree-traversal.md)
The VDB 3-4-5 Tree Traversal is a method for retrieving leaf node data from an OpenVDB file stored in a compressed texture atlas format using a hierarchical 3-4-5 tree topology. It solves the problem of random access to sparse volumetric data by mapping 3D coordinates to leaf values through a fixed-depth traversal of nodes with sizes $3^3$, $4^4$, and $5^5$. It differs from standard pointer-based OpenVDB by storing the tree structure in compressed texture atlases rather than explicit pointers, and from NanoVDB by using a specific 3-4-5 hierarchy instead of a linearized or standard octree representation.

### [OpenVDB File Format Parsing](5173-openvdb-file-format-parsing.md)
This method describes the binary parsing of the OpenVDB file format to reconstruct sparse voxel grids in memory. It solves the problem of loading hierarchical, compressed volumetric data from disk into a traversable tree structure. Unlike raw voxel grids, this method handles sparse storage via Node5 and Node4 internal nodes, and Node3 leaf nodes, supporting both active mask compression and value compression. It differs from [NanoVDB Linear Representation](1527-nanovdb-linear-representation.md) by maintaining a hierarchical tree structure in memory rather than a flat linear array, and from [VDB Volume Rendering with OpenVDB/NanoVDB](804-vdb-volume-rendering-with-openvdb-nanovdb.md) by focusing solely on data loading rather than rendering traversal.

## Operating characteristics
*   **Time Complexity:** $O(1)$ random access for modification and retrieval [S1, unofficial].
*   **Memory Complexity:** Sparse storage; only non-empty nodes and active voxels are stored. The 64-bit bit masks add a fixed overhead per node [S1, unofficial].
*   **Performance:** Well-suited for visual effects rendering of volumetric data from fluid simulations [S1, unofficial]. Fast access and modification allow for runtime editing [S1, unofficial].
*   **Strengths:** Fast $O(1)$ access, out-of-core streaming support, integration with production tools (Houdini, Cycles) [S1, unofficial].
*   **Limitations:** Data prevails in CPU-visible address spaces, requiring conversion for GPU use (e.g., NanoVDB) [S1, unofficial].
*   **When to Use:** Large, sparse volumetric datasets requiring random access and modification, particularly in VFX and scientific visualization [S1, unofficial].
*   **When Not to Use:** When GPU-native memory layouts are required without conversion overhead, or for dense volumes where the sparse overhead is not justified.

## Background
OpenVDB was introduced by Ken Museth in a paper that established the sparse voxel representation [S1, unofficial]. It is an open-source library resulting from Museth's work, designed for hierarchical encoding and compression of volumetric grids [S1, unofficial]. It has become well-established due to its integration into production renderers like Cycles and visual effects software such as Houdini and Cinema 4D [S1, unofficial]. It is popular in industry-strength libraries optimized for construction and sampling on NVIDIA GPUs [S1, unofficial].

## Sources
[S1] GPU Volume Rendering with Hierarchical Compression Using VDB, https://arxiv.org/pdf/2504.04564, fetched 2026-10-26 (unofficial)
[S2] NanoVDB: A GPU-Optimized Sparse Voxel Data Structure, https://developer.nvidia.com/blog/nanovdb-a-gpu-optimized-sparse-voxel-data-structure/, fetched 2026-10-26
