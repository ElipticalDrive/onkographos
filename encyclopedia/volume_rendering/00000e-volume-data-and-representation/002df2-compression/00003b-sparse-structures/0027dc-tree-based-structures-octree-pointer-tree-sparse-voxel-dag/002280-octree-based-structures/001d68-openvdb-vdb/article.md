# OpenVDB (VDB)

Also called: OpenVDB Hierarchical Compression, VDB

## Overview
OpenVDB is a sparse voxel representation used for the hierarchical encoding and compression of volumetric grids, featuring fast $O(1)$ random access for both modification and retrieval operations [S1]. It solves the problem of efficiently storing and accessing large, sparse 3D datasets by using a shallow tree structure that balances memory overhead with traversal speed [S1]. Unlike deeper hierarchical structures that may incur significant traversal overhead, OpenVDB is optimized for shallow trees with four levels, making it particularly suitable for production rendering and visual effects where runtime modification and out-of-core streaming are required [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volumetric grid (dense or sparse) with associated scalar or vector values per voxel.
*   **Output:** A hierarchical tree structure consisting of a root node, internal nodes (tiles), and leaf nodes, along with associated metadata and voxel data arrays.

**Data Structures:**
The OpenVDB structure is a shallow tree with exactly four levels [S1].
1.  **Root Level:** Contains a variable number of children. It stores a 64-bit direct access bit mask.
2.  **Internal Nodes (Tiles):** These nodes have a fixed number of child nodes. They store a 64-bit direct access bit mask.
3.  **Leaf Nodes:** These nodes store a fixed number of voxels (default $8 \times 8 \times 8$). They store a 64-bit direct access bit mask, a bit mask for active voxels, and a raw pointer to the voxel data.

**Bit Mask Layout:**
At each level, a 64-bit value serves as a direct access bit mask encoding topology or metadata [S1]:
*   **Internal Nodes:** The bit mask encodes the child ID or other representative values for the entire tile [S1].
*   **Leaf Nodes:** The bit mask encodes the leaf origin using $3 \times 20$ bits. Remaining bits encode additional information, such as whether the voxels are compressed or quantized [S1].

**Voxel Data:**
Leaf nodes contain a bit mask indicating which voxels are active within the leaf block. A raw pointer references the array of voxel values corresponding to the active voxels [S1].

**Parameters:**
*   **Leaf Node Size:** Default is $8 \times 8 \times 8$ voxels [S1].
*   **Bit Mask Width:** 64 bits per node level [S1].

**Algorithm: Tree Construction and Access**
The following pseudocode demonstrates the logical structure of the OpenVDB tree and how to access a voxel. Note that this is a logical representation; actual implementation involves specific memory layouts and pointer arithmetic.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class OpenVDBNode:
    def __init__(self, level, bit_mask):
        self.level = level  # 0: Root, 1-2: Internal, 3: Leaf
        self.bit_mask = bit_mask  # 64-bit integer
        self.children = []  # List of child nodes (for internal nodes)
        self.active_voxel_mask = 0  # Bitmask for active voxels (Leaf only)
        self.voxel_data = None  # Pointer/Array to voxel values (Leaf only)
        self.origin = None  # (x, y, z) coordinates (Leaf only)

    def get_child_id(self, index):
        """
        For internal nodes, extract child ID from the 64-bit bit mask.
        Implementation detail: The specific bit extraction logic depends on 
        the internal node's specific encoding scheme defined in the OpenVDB spec.
        """
        # Placeholder for specific bit extraction logic from bit_mask
        return (self.bit_mask >> index) & 1

    def get_leaf_origin(self):
        """
        For leaf nodes, extract origin from the 64-bit bit mask.
        Uses 3x20 bits.
        """
        # Placeholder for extracting 3x20 bits from bit_mask
        return (self.bit_mask & 0x7FFFFF) # Simplified example

class OpenVDBTree:
    def __init__(self):
        self.root = OpenVDBNode(level=0, bit_mask=0)

    def set_leaf_size(self, size=8):
        self.leaf_size = size

    def access_voxel(self, x, y, z):
        """
        Access a voxel at global coordinates (x, y, z).
        Assumes a 4-level tree.
        """
        # Determine leaf coordinates based on leaf size
        leaf_x = x // self.leaf_size
        leaf_y = y // self.leaf_size
        leaf_z = z // self.leaf_size
        
        # Traverse from root to leaf
        # Note: Actual traversal uses the 64-bit bit masks to find children
        # This is a simplified logical traversal
        
        current_node = self.root
        
        # Level 1: Root to first internal level
        # Use bit mask to find child corresponding to (leaf_x, leaf_y, leaf_z)
        child_idx = self._get_child_index(current_node, leaf_x, leaf_y, leaf_z)
        current_node = current_node.children[child_idx]
        
        # Level 2: Internal to next internal level
        child_idx = self._get_child_index(current_node, leaf_x, leaf_y, leaf_z)
        current_node = current_node.children[child_idx]
        
        # Level 3: Internal to Leaf
        child_idx = self._get_child_index(current_node, leaf_x, leaf_y, leaf_z)
        leaf_node = current_node.children[child_idx]
        
        # Level 4: Leaf to Voxel
        # Calculate local index within the leaf block
        local_x = x % self.leaf_size
        local_y = y % self.leaf_size
        local_z = z % self.leaf_size
        local_index = local_x + local_y * self.leaf_size + local_z * self.leaf_size * self.leaf_size
        
        # Check if voxel is active using active_voxel_mask
        if (leaf_node.active_voxel_mask >> local_index) & 1:
            return leaf_node.voxel_data[local_index]
        else:
            return None  # Or background value

    def _get_child_index(self, node, x, y, z):
        """
        Determine child index based on coordinates and node level.
        This is a placeholder for the specific Morton-order or bit-mask 
        decoding logic used in OpenVDB.
        """
        # Placeholder logic
        return 0
```

**Edge Cases and Pitfalls:**
*   **Precision:** The use of 64-bit bit masks limits the maximum tree depth and coordinate range. For very large domains, the origin encoding in leaf nodes ($3 \times 20$ bits) may require careful handling to avoid overflow or misalignment [S1].
*   **Sparsity:** If the volume is not sparse, the overhead of the tree structure and bit masks may outweigh the benefits. OpenVDB is designed for sparse data [S1].
*   **Modification:** While OpenVDB supports fast $O(1)$ random access for modification, frequent structural changes (adding/removing nodes) may require rebalancing or reallocation, which can be costly [S1].

## Subtopics

### [VDB Volume Rendering with OpenVDB/NanoVDB](804-vdb-volume-rendering-with-openvdb-nanovdb.md)
This method enables GPU volume rendering of large, sparse volumetric datasets by combining offline fixed-rate lossy compression on the CPU (using OpenVDB) with fast, low-overhead on-the-fly decompression and sampling on the GPU (using NanoVDB). It solves the problem of rendering volumes that exceed single-GPU memory capacity by significantly reducing the memory footprint while maintaining random access patterns suitable for incoherent workloads like path tracing. Unlike hierarchical space-skipping approaches which show diminishing returns with RGBA transfer functions, this method typically employs a uniform majorant grid for traversal acceleration to facilitate direct comparison with dense texture rendering.

### [NanoVDB Linear Representation](1527-nanovdb-linear-representation.md)
NanoVDB is a linear, GPU-optimized implementation of the OpenVDB sparse voxel structure. It solves the problem of efficiently transferring and sampling large, sparse volumetric datasets on the GPU by eliminating CPU-dependent memory pointers and enabling compact memory copying via `memcpy`. Unlike its parent [OpenVDB (VDB)](1508-openvdb-vdb.md), which supports complex runtime modifications, NanoVDB is designed as an immutable "linear snapshot" optimized strictly for fast retrieval and on-the-fly decompression during rendering.

### [VDB 3-4-5 Tree Traversal](4996-vdb-3-4-5-tree-traversal.md)
The VDB 3-4-5 Tree Traversal is a method for retrieving leaf node data from an OpenVDB file stored in a compressed texture atlas format using a hierarchical 3-4-5 tree topology. It solves the problem of random access to sparse volumetric data by mapping 3D coordinates to leaf values through a fixed-depth traversal of nodes with sizes $3^3$, $4^4$, and $5^5$. It differs from standard pointer-based OpenVDB by storing the tree structure in compressed texture atlases rather than explicit pointers, and from NanoVDB by using a specific 3-4-5 hierarchy instead of a linearized or standard octree representation.

### [OpenVDB File Format Parsing](5173-openvdb-file-format-parsing.md)
This method parses the binary OpenVDB file format to reconstruct sparse voxel grids in memory. It loads hierarchical, compressed volumetric data from disk into a traversable tree structure consisting of a root node, internal nodes, and leaf nodes. It differs from [NanoVDB Linear Representation](1527-nanovdb-linear-representation.md) by maintaining a hierarchical tree structure in memory rather than a flat linear array, and from [VDB Volume Rendering with OpenVDB/NanoVDB](804-vdb-volume-rendering-with-openvdb.md) by focusing solely on data loading rather than rendering traversal.

## Operating characteristics
*   **Time Complexity:** Random access for modification and retrieval is $O(1)$ [S1]. Traversal depth is fixed at four levels, making traversal time constant with respect to domain size, though dependent on tree topology [S1].
*   **Memory Complexity:** Memory usage is proportional to the number of non-empty voxels and the overhead of the tree structure (bit masks and pointers). Sparsity leads to significant compression [S1].
*   **Performance:** OpenVDB is well-suited for visual effects rendering of volumetric data from fluid simulation [S1]. It is optimized for construction and sampling on NVIDIA GPUs [S1].
*   **Strengths:** Fast random access, shallow tree structure, out-of-core streaming support, integration with production renderers (Cycles) and VFX software (Houdini, Cinema 4D) [S1].
*   **Limitations:** Data remains in CPU address spaces by default, requiring explicit transfer to GPU for rendering [S1]. Not optimized for deep trees or highly dense data.
*   **When to Use:** Large, sparse volumetric datasets requiring random access and modification, particularly in VFX and scientific visualization contexts [S1].
*   **When Not to Use:** Dense volumes, scenarios requiring deep hierarchical traversal, or when GPU-native immutable structures are preferred (use NanoVDB instead) [S1].

## Background
OpenVDB was introduced by Ken Museth in a paper published in 2013 [S1]. It is an open-source library that has become well-established in the industry due to its integration into production renderers like Cycles and visual effects software such as Houdini and Cinema 4D [S1]. The structure is named VDB because it represents volumetric grids that share similarities with B+trees [S1]. It is particularly popular in production rendering and has libraries optimized for construction and sampling on NVIDIA GPUs [S1].

## Sources
[S1] GPU Volume Rendering with Hierarchical Compression Using VDB, https://arxiv.org/pdf/2504.04564, fetched 2026-10-26
