# Variable Bit Length Encoding

Also called: 可变字长编码

## Overview
Variable Bit Length Encoding (VBLE) is a lossless compression technique for volume data that reduces storage by encoding voxel values relative to a base value within a data block (brick). It solves the problem of inefficient fixed-width storage when data values in a local region have a small dynamic range [S1, unofficial]. Unlike its parent [Variable-Length Coding](7012-variable-length-coding.md), which determines a single bit depth $n$ for the entire brick based on the global range, VBLE typically employs a hierarchical structure (such as a quadtree) to determine bit lengths locally, allowing finer granularity in compression [S1, unofficial].

## How it works
**Inputs:** A volume dataset, typically processed as a collection of 2D slices or 3D bricks.
**Outputs:** A compressed bitstream containing base values, bit-length indicators, and variable-length residuals.

**Data Structures:**
1.  **Quadtree:** For each data slice, a quadtree structure is built to partition the data into regions of uniform value range [S1, unofficial].
2.  **Bitstream:** Stores the encoded residuals and metadata.

**Steps:**
1.  **Partitioning:** Divide the volume data into blocks (bricks) or slices. For slice-based approaches, build a quadtree for each slice [S1, unofficial].
2.  **Base Value Selection:** For each block or leaf node in the quadtree, identify the minimum value $v_{min}$ within that region [S1, unofficial].
3.  **Residual Calculation:** Compute the difference (residual) for each voxel $v_i$ in the region:
    $$ r_i = v_i - v_{min} $$
    [S1, unofficial]
4.  **Bit Length Determination:** Determine the number of bits $n$ required to represent the maximum residual in the region. If $R_{max}$ is the maximum residual:
    $$ n = \lceil \log_2(R_{max} + 1) \rceil $$
    [S1, unofficial]
5.  **Encoding:**
    *   Store the base value $v_{min}$ using a fixed or standard encoding.
    *   Store the bit length $n$ for the block. *Note:* VBLE usually needs to be combined with Run-Length Encoding (RLE) to efficiently store the encoding bit length of each voxel or block, especially when many blocks share the same bit length [S1, unofficial].
    *   Encode each residual $r_i$ using exactly $n$ bits [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Overhead:** Storing the bit length for every voxel or block can incur substantial overhead relative to the value bits themselves [S2].
*   **Precision:** Ensure that the bit length $n$ is sufficient to represent the maximum residual without overflow.
*   **Structure Choice:** The choice between block-based and quadtree-based partitioning affects compression ratio and decoding complexity [S1, unofficial].

## Subtopics

### RBUC Decoder Implementation
RBUC (Recursive Bit-Universal Coding) is a variant that amortizes length bits over several consecutive residuals using a tree structure [S2]. The decoder uses pre-order traversal of the RBUC tree, which is replaced by three nested loops (one per level) maintaining the 'current' bit length on each level [S2]. Loops are unrolled in GPU implementations for conditional-free code [S2]. The decoder determines the number of bits to read for the next few nodes and fetches variable-length residuals from the bit stream [S2]. Signed residuals are stored biased by $2^{w-1}$ to move them into the unsigned range $[0, 2^w - 1]$ [S2].

### RBUC Residual Coding
RBUC is a variable-length lossless coding scheme for residuals [S2]. It exploits coherence by amortizing length bits over several consecutive residuals using a tree structure [S2]. Each interior node stores $w = \max_i(\lceil \log_2(x_i + 1) \rceil)$ over child nodes with unsigned values $x_i$ [S2]. Residuals are mapped to unsigned ordinals (e.g., $0, -1, +1, -2, +2, \dots$) for computing $w$ [S2]. Trees are shallow (4 levels for residuals up to 127) [S2]. Static branching is used: lowest-level branching factor 4-5 vertices, second-level branching ~16 nodes [S2]. Trees avoid grouping residuals from the end of one row with the start of the next to preserve spatial locality [S2]. Each row is partitioned into groups of 3-5 vertices, with second-level groups spanning whole rows [S2].

## Operating characteristics
*   **Time Complexity:** Encoding is linear with respect to the number of voxels, plus the cost of building the quadtree structure [S1, unofficial]. Decoding is also linear, but RBUC variants allow for efficient, conditional-free decoding on GPUs [S2].
*   **Memory Complexity:** Requires storage for the tree structure (if used) and the compressed bitstream [S1, unofficial].
*   **Strengths:** Effective for data with local uniformity or small dynamic ranges [S1, unofficial]. RBUC variants reduce overhead by amortizing bit-length information [S2].
*   **Limitations:** Overhead from storing bit lengths can be significant if not managed (e.g., via RLE) [S1, unofficial][S2]. Not suitable for data with high entropy or large random variations within blocks [S1, unofficial].
*   **When to use:** Medical imaging and scientific volume data where background voxels have small values and non-background voxels have larger values, allowing for efficient local encoding [S1, unofficial].

## Background
VBLE is a lossless compression technique suitable for volume data, particularly in medical imaging and scientific visualization where data is often archived losslessly [S1, unofficial]. It is often applied to 2D slices of the volume, treating them as collections of data blocks [S1, unofficial]. The technique is related to other block-based compression methods like Burrows-Wheeler Transform (BWT) but focuses on value range reduction [S1, unofficial]. Klajnsek et al. proposed using quadtree structures for slice-based VBLE [S1, unofficial]. RBUC was proposed by Moffat and Anh to improve upon traditional variable-length codes by amortizing length bits [S2].

## Sources
[S1] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
[S2] On-the-fly decompression and rendering of multiresolution terrain, https://doi.org/10.1145/1730804.1730815, fetched 2026-11-01
