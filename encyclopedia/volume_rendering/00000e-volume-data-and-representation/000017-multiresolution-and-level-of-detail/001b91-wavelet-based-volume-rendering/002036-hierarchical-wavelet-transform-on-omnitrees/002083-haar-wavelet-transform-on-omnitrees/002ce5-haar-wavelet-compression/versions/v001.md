# Haar Wavelet Compression

Also called: 3D Haar Wavelet Compression, 3D Haar Wavelet Compression with Random Access

## Overview
Haar Wavelet Compression is a 3D compression scheme for interactive visualization of very large volume data, designed to allow random access to parts of the volume without decompressing the entire dataset [S1, unofficial]. It sets itself apart from earlier wavelet compression schemes by combining a hierarchical Haar wavelet transform with a specific block-subdivision storage layout that enables efficient random access [S1, unofficial].

## How it works
**Inputs:** A 3D volume dataset where original voxel values are stored with 12 bits.
**Outputs:** Compressed data streams and metadata structures enabling random access and reconstruction.

**Data Structures and Layout:**
The method organizes data into hierarchical blocks and cells to facilitate random access.
1.  **Unit Block:** The volume is partitioned into blocks of size $16 \times 16 \times 16$.
2.  **Cells:** Each unit block is subdivided into $4^3 = 64$ subblocks, called "cells". Each cell has dimensions $4 \times 4 \times 4$.
3.  **Cell Information Chunk:** For each unit block, a metadata structure is created consisting of:
    *   A $4 \times 4 \times 4$ one-bit flag block (64 bits total). Each bit corresponds to a cell. If the bit is 1, the cell contains non-zero coefficients; if 0, all coefficients in that cell are zero.
    *   Offset information pointing to the address of the cell's coefficients in the data streams.
4.  **Coefficient Streams:** Coefficients are stored in two separate arrays based on their byte size:
    *   `one-byte`: Stores coefficients requiring 1 byte.
    *   `two-byte`: Stores coefficients requiring 2 bytes.
    *   Cells are enumerated with positive integers in increasing order if they contain non-zero coefficients. Cells with all zero coefficients are tagged with zero and do not contribute to the enumeration or data streams.

**Algorithm Steps:**

1.  **Block Partitioning:** Divide the input volume into $16 \times 16 \times 16$ unit blocks.
2.  **Initial Wavelet Transform:** Apply two passes of a three-dimensional Haar wavelet transform to each unit block. This reduces the data size and decorrelates the data [S1, unofficial].
3.  **Subblock Selection and Transform:** Within each unit block, select $2 \times 2 \times 2$ subregions. Apply a Haar wavelet transform to each subregion. This step generates the eight-band filter bank coefficients [S1, unofficial].
4.  **Quantization and Thresholding:** Quantize the resulting coefficients. Approximate the original information by keeping only coefficients whose absolute values are greater than a specific threshold value. Coefficients below the threshold are treated as zero [S1, unofficial].
5.  **Classification and Storage:**
    *   Determine the byte size (1 or 2 bytes) required to represent each non-zero coefficient.
    *   Assign each coefficient to the `one-byte` or `two-byte` stream.
    *   Update the Cell Information Chunk: Set the flag for a cell to 1 if it contains any non-zero coefficients, and 0 otherwise. Record the offset in the corresponding stream for each non-zero cell.
6.  **Reconstruction:**
    *   Use the Cell Information Chunk to identify which cells contain data.
    *   Retrieve coefficients from the `one-byte` and `two-byte` streams using the offsets.
    *   For each $4 \times 4 \times 4$ subregion (cell) within a unit block, treat it as an octree.
    *   The root node of the octree stores the average of all voxel values in the subregion (corresponding to the `clll` coefficient).
    *   Use the seven detail coefficients (`cllh`, `clhl`, `clhh`, `chll`, `chlh`, `chhl`, `chhh`) along with the root average to reconstruct the eight voxel values of the corresponding $2 \times 2 \times 2$ subregion using the inverse Haar transform equations [S1, unofficial].

**Formulas:**

The Haar wavelet transform on a $2 \times 2 \times 2$ subregion computes eight coefficients ($c_{llll}, c_{lllh}, c_{lhll}, c_{lhhl}, c_{hlll}, c_{hlhl}, c_{hhll}, c_{hhhh}$) from the eight input voxel values ($c_1, \dots, c_8$). The notation $c$ represents the coefficient, and the subscripts $l$ (low) and $h$ (high) indicate the filter applied in each dimension (x, y, z).

**Forward Transform (Decomposition):**
$$
\begin{aligned}
c_{llll} &= \frac{c_1 + c_2 + c_3 + c_4 + c_5 + c_6 + c_7 + c_8}{8} \\
c_{lllh} &= \frac{c_1 + c_2 + c_3 + c_4 - c_5 - c_6 - c_7 - c_8}{8} \\
c_{lhll} &= \frac{c_1 + c_2 - c_3 - c_4 + c_5 + c_6 - c_7 - c_8}{8} \\
c_{lhhl} &= \frac{c_1 + c_2 - c_3 - c_4 - c_5 - c_6 + c_7 + c_8}{8} \\
c_{hlll} &= \frac{c_1 - c_2 + c_3 - c_4 + c_5 - c_6 + c_7 - c_8}{8} \\
c_{hlhl} &= \frac{c_1 - c_2 + c_3 - c_4 - c_5 + c_6 - c_7 + c_8}{8} \\
c_{hhll} &= \frac{c_1 - c_2 - c_3 + c_4 + c_5 - c_6 - c_7 + c_8}{8} \\
c_{hhhh} &= \frac{c_1 - c_2 - c_3 + c_4 - c_5 + c_6 + c_7 - c_8}{8}
\end{aligned}
$$
Here, $c_i$ represents the $i$-th voxel value in the $2 \times 2 \times 2$ subregion. $c_{llll}$ represents the average of the eight voxels [S1, unofficial].

**Inverse Transform (Reconstruction):**
Given the eight wavelet coefficients, the original eight voxel values ($c_1, \dots, c_8$) are reconstructed as:
$$
\begin{aligned}
c_1 &= c_{llll} + c_{lllh} + c_{lhll} + c_{lhhl} + c_{hlll} + c_{hlhl} + c_{hhll} + c_{hhhh} \\
c_2 &= c_{llll} + c_{lllh} + c_{lhll} + c_{lhhl} - c_{hlll} - c_{hlhl} - c_{hhll} - c_{hhhh} \\
c_3 &= c_{llll} + c_{lllh} - c_{lhll} - c_{lhhl} + c_{hlll} + c_{hlhl} - c_{hhll} - c_{hhhh} \\
c_4 &= c_{llll} + c_{lllh} - c_{lhll} - c_{lhhl} - c_{hlll} - c_{hlhl} + c_{hhll} + c_{hhhh} \\
c_5 &= c_{llll} - c_{lllh} + c_{lhll} - c_{lhhl} + c_{hlll} - c_{hlhl} + c_{hhll} - c_{hhhh} \\
c_6 &= c_{llll} - c_{lllh} + c_{lhll} - c_{lhhl} - c_{hlll} + c_{hlhl} - c_{hhll} + c_{hhhh} \\
c_7 &= c_{llll} - c_{lllh} - c_{lhll} + c_{lhhl} + c_{hlll} - c_{hlhl} - c_{hhll} + c_{hhhh} \\
c_8 &= c_{llll} - c_{lllh} - c_{lhll} + c_{lhhl} - c_{hlll} + c_{hlhl} + c_{hhll} - c_{hhhh}
\end{aligned}
$$
Note: The source text contains a typo in the forward transform for $c_{hhll}$ and $c_{hhhh}$ in the provided notes (`+ 4` instead of `+ c4`), but the inverse transform and the standard Haar definition confirm the signs. The inverse equations listed in the source are consistent with the forward transform provided [S1, unofficial].

**Parameters:**
*   **Threshold Value:** A threshold is applied to coefficients to achieve compression. The specific value is not given in the sources, but it determines which coefficients are kept [S1, unofficial].
*   **Bit Depth:** The source assumes original voxel values are 12 bits. Averages fall between 0 and $2^{12}$. Details fall between $-((2^{12} - 1)/2)$ and $((2^{12} - 1)/2)$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero Coefficients:** Cells with all zero coefficients are tagged with zero and excluded from the data streams, saving space. The flag block must be correctly maintained to identify these cells [S1, unofficial].
*   **Precision:** The reconstruction equations assume exact arithmetic. In practice, quantization errors from the thresholding step will introduce approximation errors [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The data reconstruction time is imperceptible compared to the time taken by rendering tasks [S1, unofficial].
*   **Memory Complexity:** The method uses two arrays (`one-byte`, `two-byte`) and a small metadata chunk (flag block and offsets) per unit block. The size depends on the number of non-zero coefficients after thresholding.
*   **Performance:** The scheme achieves interactive navigation of large volumes (e.g., human body data) [S1, unofficial].
*   **Strengths:** Enables random access to parts of the volume without decompressing the whole dataset. Fast reconstruction.
*   **Limitations:** Compression ratio depends on the threshold and the nature of the data. The fixed block size ($16 \times 16 \times 16$) may not be optimal for all data types.
*   **When to use:** Interactive visualization of very large volume data where random access is required.
*   **When not to use:** When lossless compression is strictly required and the thresholding step introduces unacceptable error.

## Background
Haar Wavelet Compression was presented by Insung Ihm and Sanghun Park [8]. It addresses problems from earlier work (Gute et al. [6]) which proposed wavelet compression but did not allow access to parts of the volume without decompressing the whole dataset [S1, unofficial]. The method is used with raycasting or splatting as volume rendering techniques [S1, unofficial].

## Sources
[S1] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
[S2] PDF A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
