# Offset Compression
## Overview
Offset Compression is a technique for compressing tree offsets in sparse voxel structures [S1]. It exploits the fact that the offset from a node to its first child is always $+1$, allowing it to be stored implicitly, and that offsets are typically small in lower tree levels due to depth-first assignment [S1]. It sets itself apart from general entropy encoding by using fixed-width, level-adaptive bit-packing for offsets, which aligns with GPU texture lookup granularity [S1].
## How it works
**Inputs and Outputs:**
- Input: A sparse voxel tree or DAG structure containing nodes with child pointers and sibling offsets [S1].
- Output: A compressed representation where offsets use level-adaptive bit-packing and the first-child offset is implicit [S1].

**Data Structures:**
The method modifies the storage of the offset field within a node [S1]. Instead of a fixed-size integer, the offset is stored in a variable number of bits determined by the node's depth level [S1].
- **First Child Offset:** Implicitly defined as $+1$ [S1].
- **Sibling Offset:** Stored in $B_L$ bits, where $L$ is the tree level [S1]. $B_L$ is the smallest integer byte-aligned value that can represent the maximum offset at level $L$ [S1].

**Steps:**
1. **Analyze Tree Levels:** Iterate through all nodes in the tree to find the maximum offset value for each level [S1].
2. **Calculate Bit Depth:** Determine the minimum number of bits required to encode offsets for that specific level [S1].
3. **Round to Bytes:** Round the calculated bit count up to the nearest byte for GPU performance, as a texture lookup on the GPU retrieves at least a single byte [S1].
4. **Store Offsets:** For the first child of any node, the offset is implicitly $+1$ [S1]. For subsequent children, store the offset relative to the previous child using the calculated byte-aligned bit width [S1].

**Parameters:**
- **Offset Size:** In typical examples, a two-byte offset is sufficient for the lowest five levels [S1]. Four or five bytes are required for the highest levels [S1].
- **Level Thresholds:** The highest levels (near the root) contain very few nodes ($\approx 0.1\%$), making the increased bit-width non-critical for overall memory usage [S1].

**Edge Cases and Pitfalls:**
- **Implicit First Child:** The assumption that the first child offset is always $+1$ relies on a specific depth-first assignment order [S1].
- **GPU Performance:** Failing to round up to byte boundaries can lead to inefficient memory access patterns on GPUs, where texture lookups typically retrieve at least one byte [S1].
## Subtopics

### DAG Offset Compression
A variant of Offset Compression applied to Directed Acyclic Graph (DAG) structures where attribute decoupling introduces additional offsets [S1]. It compresses both DAG offsets and child pointers to reduce memory consumption [S1]. The method stores the offset from a node to its first child implicitly as $+1$, analyzes each level to determine the minimum bit depth, and rounds up to bytes for GPU performance [S1]. In examples, a two-byte offset is sufficient for the lowest five levels, while four or five bytes are required for the highest levels, which represent $\approx 0.1\%$ of nodes [S1].

## Operating characteristics
**Time and Memory Complexity:**
- **Memory:** Reduces the storage size of offsets from a fixed width to a level-dependent width, leading to a significant improvement in typical examples [S1].
- **Time:** The compression process requires a single pass to analyze levels and calculate bit depths [G].

**Measured Performance:**
- In typical datasets, using two-byte offsets for the lowest five levels leads to significant memory improvement [S1].
- The highest levels require 4-5 bytes, but since they represent only $\approx 0.1\%$ of nodes, the impact on total memory is minimal [S1].

**Strengths and Limitations:**
- **Strengths:** Simple to implement, efficient for GPU access due to byte-aligned storage, significant memory reduction for deep trees [S1].
- **Limitations:** Less effective for pointer compression compared to offset compression, as pointers often require full 4 bytes even at lower levels [S1].

**When to Use:**
- Use when memory bandwidth is a bottleneck and the tree structure follows a depth-first assignment [S1].
- Ideal for GPU-based rendering where texture lookups benefit from byte-aligned data [S1].

**When Not to Use:**
- Not given in the sources.
## Background
Offset Compression was introduced in the context of geometry and attribute compression for voxel scenes, specifically to reduce the memory consumption of DAG structures where attribute decoupling increases the size of the structure [S1]. It builds on the observation that offsets in depth-first trees are typically small in lower levels, allowing for efficient bit-packing [S1].
## Sources
[S1] Geometry and Attribute Compression for Voxel Scenes, https://timothykol.com/pub/compression.pdf, fetched 2026-10-27
