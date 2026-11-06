# SVO Node Encoding

Also called: Octree Node Encoding, SVO Binary Format, SVO file format, SVO node structure, Sparse Voxel Octree (SVO) Binary Format, octree layout, raw_val format

## Overview
SVO Node Encoding is a compact binary representation for Sparse Voxel Octree (SVO) structures, designed to minimize memory footprint by storing both topology and attribute data within a single 32-bit integer per node [S1, unofficial]. It solves the problem of efficient storage and traversal of sparse volumetric data by using the Most Significant Bit (MSB) as a discriminator between leaf values and internal node pointers [S1, unofficial]. It differs from sibling methods like [Sparse Voxel Octree Attribute Encoding (Neighbor)](2641-sparse-voxel-octree-attribute-encoding-neighbor.md) by integrating attribute data directly into the node structure rather than separating it, and from [Packed Tensor Representation](1611-packed-tensor-representation.md) by using a hierarchical tree structure rather than a dense tensor grid.

## How it works
**Inputs:** A Sparse Voxel Octree structure containing leaf nodes (with color/density) and internal nodes (with children).
**Outputs:** A contiguous binary file stream of `uint32_t` values.

**Data Structures:**
The file consists of a header followed by the octree data array.
1.  **Header:** A sequence of `uint32_t` fields:
    *   `version`: Format version (expected `0x01`) [S1, unofficial].
    *   `volume_size.x`, `volume_size.y`, `volume_size.z`: Dimensions of the volume [S1, unofficial].
    *   `octree_resolution`: Depth of the octree [S1, unofficial].
    *   `octree_bytesize`: Total bytes reserved for the octree structure [S1, unofficial].
2.  **Octree Array:** A contiguous array of `uint32_t` values representing nodes. The sources describe the layout as a sequence of levels, where each level is allocated contiguously after the previous one, with each level consisting of 8 `uint32_t` entries representing nodes at that depth [S1, unofficial]. *Note: This "8 entries per level" description implies a specific traversal or storage order, likely corresponding to the children of the root or a specific serialization order, but the core encoding logic applies to every `uint32_t` node entry regardless of position.*

**Node Encoding Layout:**
Each node is stored as a single `uint32_t` value. The encoding relies on the Most Significant Bit (bit 31):

1.  **Null Node:** If the value is `0`, it represents a null node (no children and no data) [S1, unofficial].
2.  **Leaf Node:** If the MSB is `0` (bit 31 is clear), the node is a leaf. The remaining 31 bits store the leaf's attribute value. In the reference implementation, this is a color value packed as:
    *   7 bits for Alpha
    *   8 bits for Blue
    *   8 bits for Green
    *   8 bits for Red
    *   Format: `0xA_B_G_R` (where A is the 7-bit alpha, B is 8-bit blue, etc.) [S1, unofficial].
3.  **Internal Node:** If the MSB is `1` (bit 31 is set), the node is an internal node (parent). The remaining 31 bits store the memory address (or index/offset) of its first child node in the octree buffer [S1, unofficial].

**Algorithm:**
To encode the octree into the binary format:

1.  Initialize an empty list `nodes` to store the `uint32_t` representations.
2.  Traverse the octree in the required order (sources imply a level-order or specific traversal that results in contiguous levels, but the exact traversal algorithm for generating the linear array is not fully specified beyond the level-allocation description) [S1, unofficial].
3.  For each node in the traversal:
    a.  If the node is empty/null, append `0` to `nodes`.
    b.  If the node is a leaf:
        i.  Extract color components (R, G, B, A).
        ii. Pack into 31 bits: `value = (A << 24) | (B << 16) | (G << 8) | R`.
        iii. Ensure MSB is 0.
        iv. Append `value` to `nodes`.
    c.  If the node is internal:
        i.  Determine the index/offset of the first child in the `nodes` array.
        ii. Set MSB to 1: `value = (offset & 0x7FFFFFFF) | 0x80000000`.
        iii. Append `value` to `nodes`.
4.  Write the header fields (`version`, `volume_size`, `octree_resolution`, `octree_bytesize`) to the file.
5.  Write the `nodes` array to the file as a contiguous sequence of little-endian `uint32_t` values [S1, unofficial].

**Parameters:**
*   `version`: `0x01` [S1, unofficial].
*   Color packing: 7-bit Alpha, 8-bit Blue, 8-bit Green, 8-bit Red [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Null Node vs. Zero Value:** A value of `0` is explicitly defined as a null node. If a leaf node has a color value of `0` (black, zero alpha), it must be distinguished from a null node. The sources do not explicitly state how a zero-value leaf is encoded differently from a null node, other than implying that `0` *is* the null node. This suggests that a leaf with zero attributes might need a special encoding or that the data domain excludes zero-value leaves [S1, unofficial].
*   **Pointer Size:** The 31-bit offset limits the maximum number of nodes in the octree to $2^{31}-1$. If the octree exceeds this size, the encoding will overflow [S1, unofficial].
*   **Endianness:** The format specifies little-endian `uint32_t` values. Cross-platform compatibility requires handling endianness conversion if reading on a big-endian system [S1, unofficial].

## Subtopics

### [Block Writer for SVO Serialization](2741-block-writer-for-svo-serialization.md)
The Block Writer is a utility class responsible for serializing Sparse Voxel Octree (SVO) nodes into a binary file format. It solves the problem of persisting tree topology and data by tracking the cumulative byte and element counts during the write process. It differs from its siblings, such as the [SVO Binary Serialization Format](5123-svo-binary-serialization-format.md) which defines the complete file structure and validation logic, and the [WideNodeDescriptor Binary Layout](5156-widenodedescriptor-binary-layout.md) which defines specific node packing, by focusing specifically on the stream output mechanics and state tracking of the serialization process.

### [SVO Mask Mapping](5103-svo-mask-mapping.md)
SVO Mask Mapping is a bit-mapping scheme for encoding the existence of 8 child voxels within a Sparse Voxel Octree (SVO) node, stored in a single `uint32_t` mask field. It solves the problem of efficiently tracking which octants are occupied without storing explicit pointers for empty children, enabling compact tree traversal. It differs from sibling methods like Block Writer by focusing on the bitwise layout of child existence within the node header rather than serialization block structures.

### Texel-per-Node Octree Encoding
An optimization where each octree node is stored in exactly one texture texel to maximize depth within texture size limits [S2, unofficial]. It solves the problem of fitting deep octrees into limited GPU texture memory (e.g., 4K x 4K 2D texture in DX9) [S2, unofficial]. It differs from the standard SVO Node Encoding by mapping the 32-bit node structure to texture channels: the alpha channel (`.a`) indicates if a node is inner, leaf, or empty; the red channel (`.r`) stores the offset to the first child for inner nodes [S2, unofficial]. It assumes a root node that is a unit cube and keeps child nodes in a specific order to take advantage of implied bounding boxes [S2, unofficial].

## Operating characteristics
*   **Memory Complexity:** Each node consumes exactly 4 bytes (`uint32_t`) [S1, unofficial]. This is highly compact compared to pointer-based trees which require 8 bytes for pointers plus node overhead.
*   **Time Complexity:** Encoding and decoding are linear with respect to the number of nodes, $O(N)$, where $N$ is the number of nodes in the octree [S1, unofficial].
*   **Strengths:** Extremely low memory footprint; simple bitwise operations for node type checking and value extraction; suitable for GPU storage via texture mapping (Texel-per-Node variant) [S1, S2, unofficial].
*   **Limitations:** Limited to $2^{31}-1$ nodes due to 31-bit offset; fixed 32-bit color precision (7-bit alpha); null nodes are indistinguishable from zero-value leaves without additional context [S1, unofficial].
*   **When to use:** When memory bandwidth and storage are critical constraints, and the volume data can be represented with 7-bit alpha and 8-bit RGB channels. Ideal for GPU-based ray marching where the octree is stored in textures [S1, S2, unofficial].

## Background
The SVO Node Encoding format is described in the context of the `voxelizer` library by loryruta [S1, unofficial]. It is used for serializing sparse voxel octrees for efficient storage and retrieval. The Texel-per-Node variant was discussed in the context of efficient storage for sparse octrees on GPUs, specifically targeting DirectX 9 texture limitations [S2, unofficial].

## Sources
[S1] loryruta/voxelizer, https://github.com/loryruta/voxelizer, fetched 2026-10-27 (unofficial)
[S2] Efficient storage for a sparse octree?, https://stackoverflow.com/questions/6085209/efficient-storage-for-a-sparse-octree, fetched 2026-10-26 (unofficial)
