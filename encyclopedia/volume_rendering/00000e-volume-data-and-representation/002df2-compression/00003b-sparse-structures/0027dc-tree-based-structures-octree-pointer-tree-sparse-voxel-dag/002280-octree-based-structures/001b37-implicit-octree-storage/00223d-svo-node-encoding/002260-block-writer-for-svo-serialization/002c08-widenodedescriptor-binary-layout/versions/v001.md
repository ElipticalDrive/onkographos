# WideNodeDescriptor Binary Layout

Also called: WideNode serialization

## Overview
The WideNodeDescriptor binary layout defines the on-disk or in-memory serialization format for a "wide" node in a Sparse Voxel Octree (SVO). It stores the structural topology (which children exist) and the offsets to their data, enabling efficient traversal without storing explicit pointers. It differs from standard SVO node encodings by using 64-bit masks and base offsets, allowing support for larger subtrees or denser packing, and is distinct from the sibling [SVO Binary Serialization Format](5123-svo-binary-serialization-format.md) which may use different field sizes or ordering.

## How it works
**Inputs and Outputs:**
- Input: A WideNodeDescriptor structure instance.
- Output: A 24-byte binary buffer.

**Data Structures:**
The WideNodeDescriptor consists of four fields written in strict order:
1. `child_mask`: `uint64_t` (8 bytes). A bitmask where the $i$-th bit is set if child node $i$ exists.
2. `leaf_mask`: `uint64_t` (8 bytes). A bitmask where the $i$-th bit is set if child node $i$ is a leaf.
3. `child_base`: `uint32_t` (4 bytes). The offset (in bytes or elements, depending on context) to the start of the child node data in the stream.
4. `payload_base`: `uint32_t` (4 bytes). The offset to the start of the payload data (e.g., voxel data for leaves) in the stream.

**Numbered Steps:**
1. Initialize an empty byte buffer.
2. Write `child_mask` as a little-endian (or native, if specified by the broader serialization context) 64-bit unsigned integer.
3. Write `leaf_mask` as a 64-bit unsigned integer.
4. Write `child_base` as a 32-bit unsigned integer.
5. Write `payload_base` as a 32-bit unsigned integer.

**Formulas:**
- Total size: $8 + 8 + 4 + 4 = 24$ bytes.
- Child count: $N = \text{popcount}(child\_mask)$.
- Leaf count: $L = \text{popcount}(leaf\_mask)$.

**Parameters:**
- Endianness: Not specified in the source notes. Implementers should assume the endianness of the host system or the broader serialization format's specification [G].

**Edge Cases and Pitfalls:**
- If `child_mask` is zero, the node is a leaf, and `child_base` and `payload_base` may be ignored or set to zero.
- Offsets are relative to the start of the node's data block or the file start, depending on the parent [Block Writer for SVO Serialization](2741-block-writer-for-svo-serialization.md)'s implementation.

## Subtopics

### LeafSpec Binary Layout
The LeafSpec binary layout serializes leaf node metadata. It consists of 5 fields: `coord_min.x` (int32, 4 bytes), `coord_min.y` (int32, 4 bytes), `coord_min.z` (int32, 4 bytes), `depth` (int32, 4 bytes), and `payload_index` (uint32, 4 bytes). Total size is 20 bytes per spec [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Serialization is $O(1)$ as it involves writing a fixed number of fields.
- **Memory Complexity:** $O(1)$, requiring exactly 24 bytes per wide node.
- **Strengths:** Fixed-size layout simplifies random access and parsing. 64-bit masks support up to 64 children (though octrees have 8, this allows for future extensions or sparse packing).
- **Limitations:** Larger than minimal encodings (e.g., using 8-bit masks). Not given in the sources whether offsets are relative to the node or global.

## Background
The WideNodeDescriptor is part of the sparse-voxels library by hectorpiteau. It is used in the context of SVO serialization to store node topology and data offsets efficiently.

## Sources
[S1] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
