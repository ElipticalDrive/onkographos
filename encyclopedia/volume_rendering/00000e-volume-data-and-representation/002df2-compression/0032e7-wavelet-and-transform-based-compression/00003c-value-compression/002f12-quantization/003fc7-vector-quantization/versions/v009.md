# Vector Quantization
Also called: LBG algorithm modification, 基于字典的压缩

## Overview
Vector Quantization (VQ) is a lossy compression technique that maps $n$-dimensional input vectors to a single fixed-length index referencing an entry in a pre-defined or learned codebook (dictionary) [S1, S3, unofficial][S5]. It solves the problem of high storage requirements for volumetric data by exploiting spatial redundancy, where many local data blocks exhibit similar patterns [S5]. Unlike [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) which relies on fixed mathematical bases, VQ uses data-specific dictionaries that typically achieve better compression ratios for data with high spatial coherence [S5]. Its primary advantage is extremely fast decompression, often implemented directly on the GPU via texture lookups, making it suitable for real-time rendering of large or time-varying volumes [S1, unofficial][S5].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ $n$-dimensional source vectors $S = \{\vec{x}_1, \vec{x}_2, \dots, \vec{x}_N\}$, where each $\vec{x}_i$ represents a block of volumetric data (e.g., a $4 \times 4 \times 4$ block of voxels) [S1, S3, unofficial].
*   **Codebook:** A set of $K$ codewords $C = \{\vec{c}_1, \vec{c}_2, \dots, \vec{c}_K\}$, where $K = 2^r$ and $r$ is the number of bits per index [S1, S3, unofficial].
*   **Output:** A mapping index $j \in \{0, \dots, K-1\}$ for each input vector, identifying the best-matching codeword in $C$ [S1, S3, unofficial].

**Data Structures:**
1.  **Source Vectors:** The volume is partitioned into non-overlapping blocks. Each block is flattened into a vector $\vec{x}_i$ [S5].
2.  **Codebook:** A table of $K$ vectors, each of dimension $n$. Stored in GPU texture memory for fast access [S1, unofficial][S2].
3.  **Index Map:** A data structure storing the index $j$ for each block. For sparse data, this may be a 3D texture or a compact array [S1, unofficial][S2].

**Algorithm Steps:**

1.  **Block Partitioning:** Divide the input volume $V$ into small, fixed-size data blocks (e.g., $4 \times 4 \times 4$) [S5][S6, unofficial].
2.  **Codebook Generation (Training):**
    *   Initialize $K$ codewords (e.g., using random sampling or Principal Component Analysis splitting) [S1, S3, unofficial].
    *   Iterate until convergence:
        a.  **Assignment:** For each source vector $\vec{x}_m$, find the codeword $\vec{c}_j$ that minimizes the distortion metric $\delta(\vec{x}_m, \vec{c}_j)$. The standard metric is squared Euclidean distance:
            $$ \delta(\vec{x}, \vec{y}) = \|\vec{x} - \vec{y}\|_2^2 $$
            [S3, unofficial]
        b.  **Update:** Recalculate each codeword $\vec{c}_j$ as the centroid of all source vectors assigned to it:
            $$ \vec{c}_j = \frac{\sum_{m \in p_j} \vec{x}_m}{|p_j|} $$
            where $p_j$ is the set of vectors assigned to codeword $j$ [S1, S3, unofficial].
    *   The goal is to minimize the average distortion:
        $$ D_{avg} = \frac{1}{N} \sum_{m=1}^N \|\vec{x}_m - \vec{c}_m\|^2 $$
        [S1, unofficial]
    *   *Note:* Standard iterative algorithms like LBG suffer from the "empty cell problem" (cells with no assigned vectors) and slow convergence. Enhanced splitting strategies (e.g., PCA-based) are used to avoid empty cells and speed up training [S1, S3, unofficial].
3.  **Encoding:**
    *   For each block $\vec{x}_i$, search the codebook for the entry $\vec{c}_j$ that minimizes $\|\vec{x}_i - \vec{c}_j\|^2$ [S5].
    *   Store the index $j$ (using $r$ bits) in the index map [S1, unofficial][S2].
4.  **Decoding (Rendering):**
    *   Retrieve the index $j$ for the current block [S1, unofficial].
    *   Look up the corresponding codeword $\vec{c}_j$ from the codebook [S2].
    *   Use $\vec{c}_j$ as the approximated data block for rendering [S1, unofficial].
    *   *GPU Implementation:* To overcome Cg/OpenGL limitations (no bitwise ops, float coordinates), integer block coordinates $(x,y,z)$ are mapped to texture coordinates. For a volume of size $D_x \times D_y \times D_z$:
        $$ x' = (x \cdot 0.5 + 0.5) \cdot \frac{D_x - 1}{\frac{D_x - 4}{D_y - 1}} - \frac{2 D_x - 8}{3} $$
        $$ y' = (x \cdot 0.5 + 0.5) \cdot \frac{D_y - 4}{\frac{D_y - 1}{D_z - 1}} - \frac{2 D_y - 8}{3} $$
        $$ z' = (z \cdot 0.5 + 0.5) \cdot \frac{D_z - 1}{D_z - 4} - \frac{2 D_z - 8}{3} $$
        [S1, unofficial]
        These coordinates are clamped to $[0.0, 1.0]$ using `GL_CLAMP` to handle borders [S1, unofficial]. An "Address Texture" (e.g., $4^3$ block) stores the $w$-coordinates (weights) for the codebook lookup, allowing the GPU to reconstruct the vector via texture sampling and blending [S1, unofficial].

**Parameters:**
*   **Block Size:** Commonly $4 \times 4 \times 4$ [S6, unofficial].
*   **Index Bit-size ($r$):** Controls the trade-off between quality and compression. An 8-bit index ($K=256$) is typical [S1, unofficial].
*   **Dictionary Size ($K$):** Limits achievable quality and compression rate [S5].

**Edge Cases and Pitfalls:**
*   **Empty Cell Problem:** In iterative training, some codebook entries may receive no input vectors, leading to high distortion and wasted entries [S1, S3, unofficial].
*   **Accumulated Error:** In time-varying data, decoding difference frames (P-frames) accumulates error. I-frames (original data) must be transmitted periodically to reset the state [S1, unofficial].
*   **Texture Coordinate Mapping:** GPU texture coordinates are floats, not integers. Incorrect mapping leads to sampling errors [S1, unofficial].
*   **Pre-processing Time:** Learning dictionaries for massive volumes requires significant time [S5].

## Subtopics

### [Hierarchical Vector Quantization](827-hierarchical-vector-quantization.md)
Uses multiple resolution levels (codebooks) to capture coarse and fine details, allowing direct GPU rendering and minimizing CPU-GPU transfer.

### [LBG Algorithm](840-lbg-algorithm.md)
The standard iterative algorithm (Linde-Buzo-Gray) for generating VQ codebooks by minimizing distortion.

### [Progressive Encoding of Time-Varying Sequences](959-progressive-encoding-of-time-varying-sequences.md)
Extends VQ to time-varying data by reusing codebooks across frames and using I/P-frame mechanisms to manage error accumulation.

### [Vector Quantization for Volumetric Radiance Fields](1870-vector-quantization-for-volumetric-radiance-fields.md)
Applies VQ to compress voxel color features in NeRF-like models, achieving ~1 MB storage by substituting voxel features with codebook indices.

### [Fortune's Palette Compression](4988-fortune-s-palette-compression.md)
A lossless variant for sparse data with few unique values, mapping values to a compact palette.

### [Compact3D / C3DGS / EAGLES](6139-compact3d-c3dgs-eagles.md)
Compresses 3D Gaussian Splatting attributes (SH coefficients, scale) using shared codebooks, preserving position/orientation precision.

### [Codebook Sharing](7563-codebook-sharing.md)
Reuses a single codebook across multiple tree levels or similar images to reduce memory overhead.

### [Brood-based Vector Quantization](7595-brood-based-vector-quantization.md)
Applies VQ to residuals of adaptive tree broods (e.g., octrees) to exploit spatial coherence and reduce error.

### [Residual Vector Quantization (R-VQ)](12711-residual-vector-quantization-r-vq.md)
Uses a cascade of codebooks to encode residuals, improving accuracy for geometric attributes like scale and rotation.

## Operating characteristics
*   **Time Complexity:** Encoding is $O(N \cdot K \cdot n)$ for naive search, where $N$ is the number of blocks, $K$ is codebook size, and $n$ is vector dimension. Decoding is $O(1)$ per block via lookup [S1, unofficial][S5].
*   **Memory Complexity:** Storage is $O(K \cdot n + N \cdot r)$, where $r$ is bits per index. The codebook is stored once, and indices are stored per block [S2].
*   **Performance:** Achieves high compression ratios (e.g., ~20:1) with minimal artifacts for data with high spatial redundancy [S2, S5].
*   **Strengths:** Extremely fast decompression; suitable for real-time rendering; effective for sparse data and time-varying sequences [S1, unofficial][S5].
*   **Limitations:** Lossy; dictionary training is slow for large volumes; quality is limited by dictionary size [S1, unofficial][S5].
*   **When to Use:** Large volumetric datasets with high spatial redundancy; real-time rendering scenarios; time-varying data [S1, unofficial][S5].
*   **When Not to Use:** Data with low spatial coherence; lossless compression requirements (unless sparse) [S1, unofficial][S5].

## Background
Vector Quantization was first introduced in volume rendering by [NH92, NH93] [S5]. The foundational LBG algorithm was developed by Linde, Buzo, and Gray in 1980, improving upon scalar quantizers by Max (1960) and Lloyd (1982) [S3, unofficial]. Early implementations faced performance issues due to the "empty cell problem" and slow convergence, which were later addressed by PCA-based splitting strategies [S1, S3, unofficial]. GPU implementations, such as those by Schneider et al. [SW03] and Fout et al. [FM07], enabled real-time decoding by leveraging texture mapping features [S5]. VQ remains a popular technique for compressing volumetric data, especially in the context of time-varying sequences and modern neural radiance fields [S1, unofficial][S2, S5].

## Sources
[S1] A system for real-time rendering of compressed time-varying volume data, https://doi.org/10.7939/r3041m, fetched 2026-10-26 (unofficial)
[S2] Compressing Volumetric Radiance Fields to 1 MB, https://doi.org/10.1109/cvpr52729.2023.00411, fetched 2026-10-26
[S3] Compression Domain Volume Rendering, https://www.cs.cit.tum.de/fileadmin/w00cfj/cg/Research/Publications/2003/Compression_Domain_Volume_Rendering/vis03-vq.pdf, fetched 2026-10-26 (unofficial)
[S4] A 129 FPS Full HD Real-Time Accelerator for 3D Gaussian Splatting., https://doi.org/10.1109/tvcg.2026.3683714, fetched 2026-10-27
[S5] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S6] 体数据压缩技术综述, https://www.jcad.cn/article/id/9d3ce9cd-c97c-4fab-9291-a72fbe7ba37e, fetched 2026-10-30 (unofficial)
