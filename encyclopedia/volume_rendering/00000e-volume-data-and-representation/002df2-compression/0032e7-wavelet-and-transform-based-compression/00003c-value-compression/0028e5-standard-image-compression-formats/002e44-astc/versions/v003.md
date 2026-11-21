# ASTC
Also called: ASTC Decompression, Adaptive Scalable Texture Compression

## Overview
ASTC is a flexible, hardware-supported fixed-rate lossy compression method that uses various block-size configurations to cover a wide range of bit rates [S1, S2]. It solves the problem of limited bandwidth in real-time rendering by allowing on-the-fly decompression of volumetric data directly from compressed coordinates without storing the entire decompressed volume in memory [S1, S2]. Unlike its sibling [Variable-Block Color Compression](1878-variable-block-color-compression.md), which handles segmented datasets with splitting planes, ASTC is an adaptation of Block Truncated Coding (BTC) that shares similarities with S3TC and supports hardware-accelerated random access [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Compressed bitstream representing a block of texture/volume data; block dimensions ($W \times H \times D$); block type identifier.
*   **Output:** Decompressed per-texel values (colors or scalar values) reconstructed in real-time.

**Data Structures:**
The compressed data is organized into blocks. Each block contains:
1.  **Block Type/Mode:** Identifies the compression mode and block size configuration.
2.  **Endpoints/Weights:** Quantized values used to interpolate texel values.
3.  **Indices:** Selectors that map texels to the endpoints or weight sequences.
*Note: The exact bit-packing layout for the 3D ASTC specification is not fully detailed in the provided sources, as the 3D part was undergoing evaluation for OpenGL inclusion [S1]. The general structure follows BTC adaptations [S2].*

**Steps:**
1.  **Block Selection:** The volume is divided into blocks of configurable sizes (e.g., $3\times3\times3$ to $6\times6\times6$) [S1, S2].
2.  **Compression:**
    *   Represent value sequences using a fractional number of bits per value [S1, S2].
    *   Construct per-texel color weights from sparse samples [S1, S2].
    *   Encode the block type, endpoints, and indices into a fixed-rate bitstream [S2].
3.  **Decompression (On-the-fly):**
    *   Access the compressed block at the required voxel coordinates.
    *   Decode the block type to determine the block size and mode.
    *   Retrieve the endpoints and indices from the bitstream.
    *   Interpolate the per-texel values using the fractional bit representation and sparse sample weights [S1, S2].
    *   Return the reconstructed value for the specific voxel.

**Formulas:**
*   Bit rate calculation: $bps = \frac{\text{Block Size in Bits}}{\text{Number of Voxels}}$ [S1, S2].
*   For a $6\times6\times6$ block: $bps = 0.56$ [S1, S2].
*   For a $3\times3\times3$ block: $bps = 4.74$ [S1, S2].

**Parameters:**
*   **Block Sizes:** Scalable configurations. Examples include $6\times6\times6$ and $3\times3\times3$ for 3D textures [S1, S2].
*   **Bit Rate:** Ranges from 0.56 bps (large blocks) to 4.74 bps (small blocks) [S1, S2].

**Edge Cases and Pitfalls:**
*   **3D Specification Status:** The full 3D ASTC specification was still under evaluation for OpenGL inclusion at the time of the sources, implying potential variability or lack of finalized hardware support for 3D volumes [S1].
*   **Memory:** Requires no additional memory for the entire decompressed volume, but requires hardware support for random access decompression [S1, S2].

## Subtopics

### 3D ASTC
A specific application of ASTC for volumetric data. It differs from 2D texture compression by handling 3D block structures. Choose when hardware supports 3D random access decompression. See [ASTC (method)](52-astc.md).

### BTC Adaptation
ASTC is an adaptation of Block Truncated Coding (BTC). It differs from standard BTC by supporting hardware-accelerated random access and fractional bit representation. See [BTC](link-to-btc).

## Operating characteristics
*   **Time Complexity:** Decompression is performed on-the-fly in a single render pass by the GPU [S1, S2].
*   **Memory Complexity:** Low; does not require storing the entire decompressed volume in memory [S1, S2].
*   **Measured Performance:** For 3D textures, $6\times6\times6$ blocks yield 0.56 bps, and $3\times3\times3$ blocks yield 4.74 bps [S1, S2].
*   **Scaling:** Scalable via different block-size configurations covering a wide range of bit rates [S1, S2].
*   **Strengths:** Hardware-supported, flexible block sizes, competitive with DXT, ETC2, and PVRTC [S1, S2].
*   **Limitations:** 3D specification status was uncertain at the time of writing; lossy compression [S1, S2].
*   **When to Use:** Real-time rendering of time-varying volumetric datasets where bandwidth is limited and hardware supports ASTC [S1, S2].
*   **When Not to Use:** When lossless compression is required or hardware does not support ASTC decompression [S1, S2].

## Background
ASTC was introduced by Nandakumar et al. [NLP* 12] as a flexible hardware-supported technique [S1]. It is an adaptation of BTC that shares similarities with S3TC [S2]. The method was proposed to improve over industry standards such as DXT [Bro01], ETC2 [SP07], and PVRTC [Fen03] [S1]. The 3D part of the specification was under evaluation for inclusion in next OpenGL releases as of 2012 [S1]. Fraedrich et al. [FBS07] proposed a general approach for applying sequential data compression schemes to blocks for GPU decoding, which influenced the on-the-fly decoding concept [S1, S2].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
