# Fixed-rate Block-coding (VTC/S3TC)
Also called: VTC, S3TC, 3D S3TC

## Overview
Fixed-rate block-coding techniques, specifically Value Texture Compression (VTC) and S3TC, are hardware-supported lossy compression methods that approximate voxel values by selecting linear interpolation modes from a lookup table and applying formulas over precomputed representative values [S1][S2]. They differ from general scalar quantization by providing on-the-fly hardware decoding that allows rendering directly from compressed data without storing decompressed volumes in memory [S1][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A block of $N$ voxel values (typically $4 \times 4 \times 4$ for 3D S3TC/VTC variants, though block sizes may vary by specific implementation [S1]).
*   **Output:** A compressed bitstream containing the block type, representative values (endpoints), and indices.

**Data Structures:**
The compressed block consists of:
1.  **Block Type/Mode:** An identifier selecting the interpolation mode from a predefined lookup table [S1][S2].
2.  **Representative Values:** A subset of values (e.g., two endpoints) used as anchors for interpolation [S1][S2].
3.  **Indices:** Small integer values (e.g., 2-bit or 3-bit) selecting interpolation weights from the lookup table for each voxel in the block [S1][S2].

**Steps:**
1.  **Block Decomposition:** Divide the volume data into equal-sized, non-overlapping blocks [S2].
2.  **Mode Selection:** For each block, select a linear interpolation mode from a lookup table. The selection is based on minimizing reconstruction error while satisfying criteria such as maintaining the standard deviation and average value of the original data [S2].
3.  **Compute Representative Values:** Calculate a precomputed subset of representative values (e.g., minimum and maximum, or two endpoints) for the block [S1][S2].
4.  **Index Assignment:** Assign an index to each voxel in the block. This index selects a specific interpolation weight from the lookup table associated with the chosen mode [S1][S2].
5.  **Encoding:** Store the block type, representative values, and indices in the compressed bitstream [S1][S2].

**Decoding (On-the-fly):**
1.  **Access:** The hardware decoder accesses the compressed block data directly using voxel coordinates [S1][S2].
2.  **Retrieve Mode and Values:** Extract the block type and representative values from the bitstream [S1][S2].
3.  **Interpolation:** Apply the selected linear interpolation formula using the representative values and the index retrieved for the specific voxel [S1][S2].
    *   The formula typically involves linear interpolation between the representative values weighted by the index [S1][S2].
    *   *Note:* Exact formulas depend on the specific S3TC/VTC variant (e.g., 2-bit or 3-bit indices). Common S3TC 2-bit indices map to weights $\{0, 1/3, 2/3, 1\}$ or similar discrete steps [S1][S2].
4.  **Reconstruction:** The result is the reconstructed voxel value [S1][S2].

**Parameters:**
*   **Error Tolerance (Threshold Level):** The criteria for truncating the block (e.g., standard deviation and average value maintenance) can vary per block [S2].
*   **Block Size:** Typically small blocks (e.g., $4 \times 4 \times 4$) to exploit spatial coherence [S1][S2].

**Edge Cases and Pitfalls:**
*   **Precision:** Lossy compression introduces quantization error. The error tolerance parameter controls this trade-off [S2].
*   **Boundaries:** Blocks are processed independently, which may introduce discontinuities at block boundaries if not handled by the rendering pipeline [S1][S2].
*   **Degenerate Input:** Blocks with very low variance may be truncated to a single value or minimal representation [S2].

## Operating characteristics
*   **Time Complexity:** Encoding is complex due to mode selection and optimization. Decoding is $O(1)$ per voxel, performed in hardware [S1][S2].
*   **Memory Complexity:** Compressed size is fixed per block (e.g., 64 bits for 2-bit S3TC, more for 3D variants). No decompressed data storage is required during rendering [S1][S2].
*   **Performance:** Enables real-time rendering of large volumes directly from compressed memory [S1][S2].
*   **Strengths:** Hardware acceleration, random access, no decompression memory overhead [S1][S2].
*   **Limitations:** Lossy, fixed block sizes (in basic S3TC/VTC), potential block boundary artifacts [S1][S2].
*   **When to use:** Real-time direct volume rendering on hardware supporting S3TC/VTC [S1][S2].
*   **When not to use:** When lossless compression is required or hardware support is absent [S1][S2].

## Background
*   **Origin:** VTC is an extension to 3D of S3TC (S3 Texture Compression) [S2]. S3TC was introduced by S3 Graphics [INH99, Bro01]. VTC was proposed by Craig et al. [Cra04].
*   **History:** Developed to support hardware-accelerated random access in volume rendering [S2].
*   **Usage:** Used in compressed GPU-based direct volume rendering [S1][S2].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
