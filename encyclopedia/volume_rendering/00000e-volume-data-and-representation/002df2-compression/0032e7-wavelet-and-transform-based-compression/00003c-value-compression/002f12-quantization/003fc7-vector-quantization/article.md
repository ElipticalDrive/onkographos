# Vector Quantization

Also called: LBG algorithm modification, 基于字典的压缩

## Overview
Vector Quantization (VQ) is a lossy compression technique that maps $n$-dimensional input vectors to a single fixed-length index referencing an entry in a pre-defined or learned codebook (dictionary) [S1, S3, unofficial][S5]. It solves the problem of storing large volumetric datasets by exploiting spatial redundancy: identical or similar data blocks are replaced by compact indices, allowing for extremely fast decompression on the GPU [S5]. Unlike [Scalar Quantization](5726-scalar-quantization.md), which processes values independently, VQ groups data into blocks to capture correlations, and unlike [Hierarchical Vector Quantization](827-hierarchical-vector-quantization.md), it typically employs a single resolution level codebook [S3, unofficial][S5].

## How it works
**Inputs and Outputs:**
*   **Input:** A volumetric dataset $V$ divided into $n$-dimensional blocks (vectors) $\vec{x}_i$. Typical block sizes are $4 \times 4 \times 4$ [S6, unofficial].
*   **Output:** A codebook $C$ containing $K$ codewords (where $K = 2^r$ and $r$ is the index bit-size) and a mapping index list where each index points to the best-matching codeword for a block [S1, unofficial][S2, S5].

**Data Structures:**
*   **Codebook ($C$):** A list of $K$ vectors, each of dimension $n$. Stored in memory (e.g., as a 3D texture or buffer object) [S1, unofficial][S2].
*   **Index Map:** A data structure (e.g., a 3D texture or integer array) storing the $r$-bit index for each block in the volume [S1, unofficial][S2, S5].

**Algorithm Steps:**

1.  **Block Partitioning:** Divide the input volume into non-overlapping $n$-dimensional blocks [S5].
2.  **Codebook Generation:** Train a codebook $C$ using an iterative algorithm (e.g., LBG or PCA-based splitting) to minimize distortion [S1, S3, unofficial].
    *   *Note:* The LBG algorithm is prone to the "empty cell problem" and slow convergence; PCA-based splitting is often preferred for speed and robustness [S1, S3, unofficial].
3.  **Encoding (Search):** For each input block $\vec{x}_i$, find the codeword $\vec{c}_j \in C$ that minimizes the distortion metric $\delta(\vec{x}_i, \vec{c}_j)$ [S3, unofficial].
    *   The standard metric is the squared Euclidean distance: $\delta(\vec{x}, \vec{y}) = \|\vec{x} - \vec{y}\|_2^2$ [S3, unofficial].
    *   Store the index $j$ corresponding to $\vec{c}_j$ in the index map [S5].
4.  **Decoding (Lookup):** To reconstruct the volume, retrieve the codeword $\vec{c}_j$ from the codebook using the index $j$ from the index map [S1, unofficial][S5].

**Formulas:**
*   **Average Distortion:** The quality of the codebook is measured by the average distortion over $N$ vectors:
    $$ D_{avg} = \frac{1}{N} \sum_{m=1}^N \| \vec{x}_m - \vec{c}_m \|^2 $$
    where $\vec{x}_m$ is the original vector and $\vec{c}_m$ is its approximation from the codebook [S1, unofficial].
*   **GPU Texture Coordinate Mapping (for Cg/OpenGL implementations):**
    To simulate integer division (e.g., $\lfloor x/4 \rfloor$) and block addressing using texture filtering (GL_NEAREST, GL_REPEAT) in shaders where bitwise ops are unavailable [S1, unofficial]:
    $$ x' = (x \cdot 0.5 + 0.5) \cdot \frac{D_x - 1}{\frac{D_x - 4}{D_y - 1}} - \frac{2 D_x - 8}{3} $$
    $$ y' = (y \cdot 0.5 + 0.5) \cdot \frac{D_y - 4}{\frac{D_y - 1}{D_z - 1}} - \frac{2 D_y - 8}{3} $$
    $$ z' = (z \cdot 0.5 + 0.5) \cdot \frac{D_z - 1}{D_z - 4} - \frac{2 D_z - 8}{3} $$
    where $D_{x,y,z} = 2^n$ for $n \ge 3$ [S1, unofficial].
    Address texture coordinates for block indexing:
    $$ x_A = (x \cdot 0.5 + 0.5) \cdot (D_x/4) $$
    $$ y_A = (y \cdot 0.5 + 0.5) \cdot (D_y/4) $$
    $$ z_A = (z \cdot 0.5 + 0.5) \cdot (D_z/4) $$
    [S1, unofficial].

**Parameters:**
*   **Index Bit-size ($r$):** Determines codebook size $K=2^r$. An 8-bit index ($K=256$) is common, balancing compression ratio and distortion [S1, unofficial].
*   **Block Size:** $4 \times 4 \times 4$ is a typical unit [S6, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Cell Problem:** Iterative codebook generation (like basic LBG) may result in unused codebook entries, increasing distortion and execution time [S1, S3, unofficial].
*   **Accumulated Error:** In time-varying data, reconstructing differences (P-frames) and adding them to previous frames causes error accumulation. This is mitigated by periodically sending original data (I-frames) to reset the state [S1, unofficial].
*   **Texture Coordinate Bounds:** Calculated texture coordinates must be clamped to $[0.0, 1.0]$ using GL_CLAMP to handle border issues [S1, unofficial].
*   **Pre-processing Time:** Learning dictionaries for massive volumes requires significant pre-processing time [S5].

## Subtopics

### [Hierarchical Vector Quantization](827-hierarchical-vector-quantization.md)
Uses multiple resolution levels (codebooks) to capture coarse and fine details, enabling direct GPU rendering and reducing CPU-GPU transfer.

### [LBG Algorithm](840-lbg-algorithm.md)
The standard iterative method for generating VQ codebooks by minimizing distortion, though prone to empty cells and slow speed.

### [Progressive Encoding of Time-Varying Sequences](959-progressive-encoding-of-time-varying-sequences.md)
Extends VQ to time-varying data by reusing codebooks across frames and using I/P-frame mechanisms to manage error.

### [Vector Quantization for Volumetric Radiance Fields](1870-vector-quantization-for-volumetric-radiance-fields.md)
Applies VQ to compress voxel color features in NeRF-like grids, achieving ~1 MB model sizes.

### [Fortune's Palette Compression](4988-fortune-s-palette-compression.md)
A lossless variant mapping unique values to a compact palette, suitable for data with few unique values.

### [Compact3D / C3DGS / EAGLES](6139-compact3d-c3dgs-eagles.md)
Uses VQ to compress Gaussian attributes (SH coefficients, scale) in 3D Gaussian Splatting.

### [Codebook Sharing](7563-codebook-sharing.md)
Reuses a single codebook across multiple levels or images to save memory.

### [Brood-based Vector Quantization](7595-brood-based-vector-quantization.md)
Applies VQ to residuals in adaptive trees (octrees) to exploit spatial coherence.

### [Residual Vector Quantization (R-VQ)](12711-residual-vector-quantization-r-vq.md)
Decomposes quantization error across multiple stages for higher precision in geometric attributes.

## Operating characteristics
*   **Time Complexity:** Encoding is $O(N \cdot K \cdot d)$ where $N$ is the number of blocks, $K$ is codebook size, and $d$ is vector dimension. Decoding is $O(1)$ per block (lookup) [S5].
*   **Memory Complexity:** Requires storing the codebook ($K \cdot d$) and the index map ($N \cdot r$ bits) [S1, unofficial][S2].
*   **Performance:** Decompression is extremely fast, making it suitable for real-time rendering [S5].
*   **Strengths:** High compression ratios for data with high spatial redundancy; fast GPU decoding; simple implementation [S1, unofficial][S5].
*   **Limitations:** Lossy; dictionary size limits quality; pre-processing (codebook generation) can be slow; "empty cell" issues in naive implementations [S1, S3, unofficial][S5].
*   **When to use:** Highly redundant volumetric data (e.g., sparse data, time-varying sequences with high inter-frame similarity) [S1, unofficial][S5].
*   **When not to use:** Data with low redundancy or high entropy where block matching fails to find similar patterns [S5].

## Background
Vector Quantization was first introduced in volume rendering by [NH92, NH93] [S5]. The foundational LBG algorithm was developed by Linde, Buzo, and Gray in 1980, improving upon scalar quantizers by Max (1960) and Lloyd (1982) [S3, unofficial]. It is widely used for large time-varying volumetric data due to its ability to exploit high spatial and temporal redundancy [S1, unofficial].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] Compressing Volumetric Radiance Fields to 1 MB, https://doi.org/10.1109/cvpr52729.2023.00411, fetched 2026-10-26
[S3] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
[S4] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S5] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S6] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
