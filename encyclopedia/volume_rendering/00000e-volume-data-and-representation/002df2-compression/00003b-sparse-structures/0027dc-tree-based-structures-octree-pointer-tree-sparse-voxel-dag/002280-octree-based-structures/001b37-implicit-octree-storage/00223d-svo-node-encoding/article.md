# SVO Node Encoding

Also called: Octree Node Encoding, SVO Binary Format, SVO file format, SVO node structure, Sparse Voxel Octree (SVO) Binary Format, octree layout, raw_val format

## Overview
SVO Node Encoding is a compact binary serialization format for Sparse Voxel Octrees (SVO) that encodes node types and data within a single 32-bit integer, enabling efficient memory storage and traversal of sparse volumetric data [S1, unofficial]. It distinguishes itself from sibling methods like [Packed Tensor Representation](1611-packed-tensor-representation.md) by using a flat, contiguous array of `uint32_t` values with a specific bit-packing scheme (MSB flag) rather than tensor-based or hierarchical pointer structures, prioritizing cache-friendly sequential access and minimal header overhead [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A Sparse Voxel Octree structure containing nodes with associated data (e.g., color, density) and hierarchical relationships.
- Output: A binary file containing a header followed by a contiguous array of `uint32_t` values representing the octree.

**Data Structures:**
The file consists of a header followed by the octree data. The header fields are:
1. `version`: `uint32_t`, expected value `0x01` [S1, unofficial].
2. `volume_size.x`: `uint32_t` [S1, unofficial].
3. `volume_size.y`: `uint32_t` [S1, unofficial].
4. `volume_size.z`: `uint32_t` [S1, unofficial].
5. `octree_resolution`: `uint32_t` [S1, unofficial].
6. `octree_bytesize`: `uint32_t`, the number of bytes reserved for the octree structure [S1, unofficial].
7. `octree`: The actual octree data, immediately following the header [S1, unofficial].

The `octree` data is a contiguous array of `uint32_t` values. The octree is conceptually stored as a sequence of levels, where each level is allocated contiguously after the previous one [S1, unofficial]. Each level consists of entries representing nodes at that depth.

**Node Encoding Layout:**
Each node in the octree array is represented by a single `uint32_t` value. The Most Significant Bit (MSB, bit 31) acts as a type flag [S1, unofficial]:
- **Internal Node (Parent):** If MSB is 1 (`bit 31 = 1`), the node is an internal node. The remaining 31 bits store the index (offset) of the first child node in the octree array [S1, unofficial].
- **Leaf Node:** If MSB is 0 (`bit 31 = 0`), the node is a leaf. The remaining 31 bits encode the leaf's data. For color data, this is packed as: 7 bits for alpha, 8 bits for blue, 8 bits for green, 8 bits for red (ABGR packing) [S1, unofficial].
- **Empty Node:** A value of `0` indicates a null node, meaning no children and no data [S1, unofficial].

**Formulas:**
Let $N$ be the `uint32_t` value of a node.
- If $N \neq 0$:
  - If $N \gg 31 == 1$: Node is internal. Child index $C = N \ \& \ 0x7FFFFFFF$.
  - If $N \gg 31 == 0$: Node is leaf. Data $D = N \ \& \ 0x7FFFFFFF$.
- If $N == 0$: Node is empty/null.

**Parameters:**
- `version`: `0x01` [S1, unofficial].
- Max texture size for variants: 4K x 4K 2D texture (DX9) [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_svo_node(is_internal, child_index_or_data):
    """
    Encodes a single SVO node into a uint32_t.
    
    Args:
        is_internal (bool): True if the node is an internal node, False if leaf.
        child_index_or_data (int): 
            If is_internal is True, this is the index of the first child in the octree array.
            If is_internal is False, this is the packed leaf data (e.g., ABGR color).
            
    Returns:
        int: The encoded 32-bit integer.
    """
    if is_internal:
        # MSB is 1, remaining 31 bits store child index
        # Ensure child_index fits in 31 bits
        if child_index_or_data < 0 or child_index_or_data > 0x7FFFFFFF:
            raise ValueError("Child index out of range for 31 bits")
        return (1 << 31) | child_index_or_data
    else:
        # MSB is 0, remaining 31 bits store leaf data
        # Ensure data fits in 31 bits
        if child_index_or_data < 0 or child_index_or_data > 0x7FFFFFFF:
            raise ValueError("Leaf data out of range for 31 bits")
        return child_index_or_data

def decode_svo_node(node_value):
    """
    Decodes a single SVO node from a uint32_t.
    
    Args:
        node_value (int): The encoded 32-bit integer.
        
    Returns:
        dict: A dictionary with keys 'type' ('empty', 'internal', 'leaf') and 'value'.
              'value' is the child index for internal nodes or the leaf data for leaf nodes.
    """
    if node_value == 0:
        return {'type': 'empty', 'value': None}
    
    msb = (node_value >> 31) & 1
    remaining_bits = node_value & 0x7FFFFFFF
    
    if msb == 1:
        return {'type': 'internal', 'value': remaining_bits}
    else:
        return {'type': 'leaf', 'value': remaining_bits}

def serialize_svo_header(volume_size, octree_resolution, octree_bytesize):
    """
    Serializes the SVO header.
    
    Args:
        volume_size (tuple): (x, y, z) dimensions of the volume.
        octree_resolution (int): Resolution of the octree.
        octree_bytesize (int): Number of bytes reserved for the octree structure.
        
    Returns:
        bytes: The serialized header.
    """
    import struct
    # Little-endian uint32_t for each field
    header = struct.pack('<IIIII', 
                         0x01, # version
                         volume_size[0],
                         volume_size[1],
                         volume_size[2],
                         octree_resolution
                         )
    header += struct.pack('<I', octree_bytesize)
    return header
```

**Edge Cases and Pitfalls:**
- **Precision:** The leaf data is limited to 31 bits. For color, this means 7 bits for alpha, which may reduce precision compared to standard 8-bit channels [S1, unofficial].
- **Boundaries:** Child indices must fit within 31 bits. If the octree is too large, the index may overflow [S1, unofficial].
- **Degenerate Input:** An empty node is represented by `0`. This must be distinguished from a valid leaf node with data `0` if such a value is possible (though in the ABGR packing, `0` implies all channels are 0, which is a valid color, but the spec defines `0` as empty, so a leaf with all-zero data might be ambiguous or require a specific convention; the source states "A value of 0 indicates a null node", implying no leaf can have value 0, or 0 is reserved for empty) [S1, unofficial].

## Subtopics

### [Block Writer for SVO Serialization](2741-block-writer-for-svo-serialization.md)
The Block Writer is a class that serializes Sparse Voxel Octree (SVO) nodes to a binary file while tracking cumulative byte and element counts. It differs from sibling methods like [SVO Mask Mapping](5103-svo-mask-mapping.md) by focusing on the serialization block structures rather than the bitwise layout of child existence within the node header.

### [SVO Mask Mapping](5103-svo-mask-mapping.md)
SVO Mask Mapping is a bit-mapping scheme for encoding the existence of 8 child voxels within a Sparse Voxel Octree (SVO) node, stored in a single `uint32_t` mask field. It solves the problem of efficiently tracking which octants are occupied without storing explicit pointers for empty children, enabling compact tree traversal. It differs from sibling methods like Block Writer by focusing on the bitwise layout of child existence within the node header rather than serialization block structures.

### [Adaptive Octree Encoding](9453-adaptive-octree-encoding.md)
Adaptive Octree Encoding is a memory layout for storing Tree-Based Adaptive Mesh Refinement (TB-AMR) data as an adaptive-resolution octree to preserve hierarchy information, enabling faster ray traversal and higher quality rendering than flattening to unstructured or rectilinear grids. Unlike Sparse Voxel Octrees (SVO) which typically store uniform voxels, this structure supports storing voxels of different resolutions to handle AMR data. It builds on [SVO Node Encoding](2705-svo-node-encoding.md) by extending the concept of compact node descriptors to support variable refinement levels inherent to AMR simulations.

### Texel-per-Node Octree Encoding
An optimization where each octree node is stored in exactly one texture texel to maximize depth within texture size limits [S2, unofficial]. It assumes a root node that is a unit cube, uses the alpha channel (.a) to indicate if a node is inner, leaf, or empty, and stores an offset to the first child in the red channel (.r) for inner nodes [S2, unofficial]. It keeps child nodes in a specific order and takes advantage of implied bounding boxes [S2, unofficial]. The max texture size is 4K by 4K 2D texture in DX9 [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Encoding/decoding is $O(1)$ per node, as it involves simple bitwise operations [S1, unofficial].
- **Memory Complexity:** Each node uses 32 bits (4 bytes) [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** The format scales with the number of nodes in the octree. The flat array structure is cache-friendly [S1, unofficial].
- **Strengths:** Compact storage, simple encoding/decoding, efficient for sparse data [S1, unofficial].
- **Limitations:** Limited precision for leaf data (31 bits), potential index overflow for very large octrees [S1, unofficial].
- **When to use:** When memory efficiency and simple traversal are prioritized, and the octree size fits within 31-bit indices [S1, unofficial].
- **When not to use:** When higher precision for leaf data is required, or when the octree is too large for 31-bit indices [S1, unofficial].

## Background
The SVO Node Encoding format is described in the context of the `loryruta/voxelizer` project, which provides a library for voxelizing 3D models into sparse voxel octrees [S1, unofficial]. The format is used to serialize the octree structure for efficient storage and retrieval [S1, unofficial]. The Texel-per-Node Octree Encoding variant is discussed in the context of efficient storage for sparse octrees on GPUs, specifically targeting DirectX 9 limitations [S2, unofficial].

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S2] Efficient storage for a sparse octree?, https://stackoverflow.com/questions/6085209/efficient-storage-for-a-sparse-octree, fetched 2026-10-26 (unofficial)
