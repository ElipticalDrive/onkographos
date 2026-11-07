# ASTC
Also called: ASTC Decompression, Adaptive Scalable Texture Compression

## Overview
ASTC is a fixed-rate lossy compression method that adapts to different block-size configurations to cover a wide range of bit rates, allowing for hardware-supported random access without storing the full decompressed volume in memory [S1, S2]. It distinguishes itself from siblings like [Variable-Block Color Compression](1878-variable-block-color-compression.md) by supporting scalable block sizes and representing value sequences using a fractional number of bits per value, enabling efficient on-the-fly reconstruction for direct volume rendering [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volumetric dataset, partitioned into blocks of configurable dimensions (e.g., $3 \times 3 \times 3$ to $6 \times 6 \times 6$).
*   **Output:** Compressed bitstream containing block type, endpoints, and indices; decompressed voxel values reconstructed in real-time from coordinates [S1, S2].

**Data Structures:**
The compressed format stores a block type identifier, endpoints (base values), and indices (weights). The specific bit-packing layout for endpoints and indices is not detailed in the provided sources [S1, S2]. The method supports different block-size configurations, where the block size determines the bit rate [S1, S2].

**Steps:**
1.  **Block Partitioning:** Divide the 3D volume into blocks of a chosen configuration (e.g., $3 \times 3 \times 3$ or $6 \times 6 \times 6$) [S1, S2].
2.  **Compression:** Encode each block using a fixed-rate lossy scheme that represents value sequences using a fractional number of bits per value [S1, S2].
3.  **Decompression (On-the-fly):** For a given voxel coordinate, identify the containing block. Retrieve the block's endpoints and indices from the compressed stream [S1, S2].
4.  **Reconstruction:** Construct per-texel color weights from sparse samples using the stored indices and endpoints to interpolate the voxel value [S1, S2].

**Formulas:**
The bit rate (bps) is determined by the block size. For 3D textures:
*   $6 \times 6 \times 6$ blocks result in $0.56$ bps [S1, S2].
*   $3 \times 3 \times 3$ blocks result in $4.74$ bps [S1, S2].

The value reconstruction involves constructing per-texel color weights from sparse samples [S1, S2]. The exact interpolation formula is not provided in the sources [S1, S2].

**Parameters:**
*   **Block Sizes:** Configurable. Examples include $6 \times 6 \times 6$ and $3 \times 3 \times 3$ [S1, S2].
*   **Bit Rates:** Vary with block size, ranging from $0.56$ bps to $4.74$ bps for the examples given [S1, S2].

**Edge Cases and Pitfalls:**
*   **Hardware Support:** The 3D part of the full ASTC specification was undergoing evaluation for inclusion in OpenGL releases as of 2012, implying potential limitations in hardware support for 3D ASTC at the time of the sources [S1, S2].
*   **Precision:** The method uses a fractional number of bits per value, which may introduce quantization errors [S1, S2].

## Operating characteristics
*   **Time Complexity:** Decompression is performed in real-time on the GPU in a single render pass, allowing direct access to voxel coordinates without full decompression [S1, S2].
*   **Memory Complexity:** Does not require additional memory to store the decompressed data, as the entire volume is not decompressed at once [S1, S2].
*   **Performance:** Results are claimed to be competitive with advanced hardware-supported formats like DXT, ETC2, and PVRTC [S1, S2].
*   **Strengths:** Scalable block sizes, hardware support, low memory footprint for rendering [S1, S2].
*   **Limitations:** 3D hardware support was not fully standardized at the time of the sources [S1, S2].
*   **When to Use:** Real-time rendering of time-varying volumetric datasets where bandwidth is limited and on-the-fly decompression is required [S1, S2].
*   **When Not to Use:** When full decompressed volume access is needed or when hardware support for the specific block sizes is unavailable [S1, S2].

## Background
ASTC was introduced by NLP* in 2012 [S1, S2]. It is an adaptation of Block Truncated Coding (BTC) that supports hardware-accelerated random access, sharing similarities with S3TC [S2]. It was designed to improve upon industry standards such as DXT, ETC2, and PVRTC [S1, S2]. The method was proposed for inclusion in OpenGL, though the 3D specification was still under evaluation at the time of the sources [S1, S2].

## Sources
[S1] State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S2] A Survey of Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-7f10-48e5-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
