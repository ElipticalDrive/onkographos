# Video-based GSC Anchor (TMCV)

Also called: GSC AhG TMCV, GSC Anchor, MPEG GSC anchor

## Overview
The Video-based GSC Anchor (TMCV) is a baseline compression framework for 3D Gaussian Splatting (3DGS) that serves as the reference implementation for the ongoing GSC standardization activity [S1]. It solves the problem of efficiently storing trained 3DGS models by combining PLAS-based spatial reordering with 2D video coding (HEVC) to compress attribute maps [S1]. It sets itself apart from sibling methods like HAC++ by relying on standard video codecs and explicit attribute grids rather than anchor-based hash grids and mutual information modeling [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A trained 3DGS model consisting of $N$ Gaussians.
*   **Output:** A set of compressed HEVC bitstreams representing the 3DGS attributes.

**Data Structures:**
Each Gaussian $i$ has 59 attributes [S1]:
*   **Position:** 3 values ($x, y, z$).
*   **Scale:** 3 values ($s_x, s_y, s_z$).
*   **Rotation:** 4 values (quaternion $w, q_x, q_y, q_z$).
*   **Opacity:** 1 value ($\alpha$).
*   **SH-DC:** 3 values ($f_{dc}$), representing RGB color.
*   **SH-AC:** 45 values ($f_{ac}$), representing view-dependent color variations [S1].

These attributes are mapped into 2D grids (attribute maps). The reordering process ensures that spatially neighboring Gaussians are placed at adjacent positions in these 2D maps [S1].

**Steps:**
1.  **PLAS Reordering:** Apply PLAS (Position and SH-DC based reordering) to the Gaussians. The permutation generated from position and SH-DC coefficients is uniformly applied to all 59 attribute grids [S1].
2.  **Position Encoding:**
    *   Quantize position attributes ($x, y, z$) to 16 bits [S1].
    *   Pack into a video sequence.
    *   Compress using HEVC in **lossless mode** to preserve geometric accuracy [S1].
3.  **Non-Position Attribute Encoding:**
    *   For all other attributes (Scale, Rotation, Opacity, SH-DC, SH-AC), perform min-max normalization [S1].
    *   Quantize to 10 bits [S1].
    *   Pack into attribute-specific video sequences using **YUV444** or **YUV400** format [S1].
4.  **HEVC Compression:**
    *   Compress all attributes into **seven** video sequences [S1].
    *   Assign Quantization Parameters (QPs) to non-position attributes based on their relative importance to support efficient bitrate allocation [S1].
    *   SH-AC attributes are encoded using standard HEVC prediction and quantization [S1].
5.  **Decoding (Reconstruction):**
    *   Decode the HEVC bitstreams [S1].
    *   Dequantize and apply inverse normalization to the attribute maps [S1].
    *   Convert the reconstructed maps back into Gaussians for rendering [S1].

**Parameters:**
*   **Position Bit-depth:** 16 bits [S1].
*   **Non-Position Bit-depth:** 10 bits [S1].
*   **Video Format:** YUV444 or YUV400 for non-position attributes [S1].
*   **QP Settings:** Assigned to all attributes except SH-AC to ensure fair visual comparison with proposed methods [S1].

**Pitfalls and Edge Cases:**
*   **Low Rate Performance:** At lower Rate Points (e.g., Bartender RP1), the anchor's decoding time is faster than proposed VQ-based methods because it processes a simpler SH-AC bitstream, whereas inverse-VQ reconstruction costs remain constant [S1].
*   **Sharp Boundaries:** In regions with sharp color boundaries (e.g., curtain and wall), the anchor maintains colors more consistently than VQ-based methods, resulting in smaller residuals in those specific areas [S1]. This is because PLAS arranges coefficients spatially, allowing HEVC spatial prediction to exploit local correlation [S1].

## Subtopics

### [VQ-HEVC Hybrid Compression for SH-AC](19574-vq-hevc-hybrid-compression-for-sh-ac.md)
Replaces the direct HEVC coding of SH-AC attribute maps with Vector Quantization (VQ) to target low spatial correlation, while keeping the anchor procedure for other attributes.

## Operating characteristics
*   **Complexity:** The anchor encodes a fixed number of SH-AC attribute frames using HEVC prediction and quantization [S1].
*   **Performance (Bartender Dataset, Intel Core i9-14900K, 62 GiB RAM, Ubuntu 20.04.6 LTS, no GPU):**
    *   **Encoding Time:** The proposed VQ method achieves lower total encoding time at all Rate Points (RP) despite codebook training overhead. However, the anchor's encoding time increases with RP [S1].
    *   **Decoding Time:** The anchor's decoding time decreases at lower RPs. At Bartender RP1, the anchor is faster than the proposed method. At higher RPs, the proposed method is comparable or faster [S1].
    *   **Memory:** Peak memory usage at Bartender RP4 was ~2.31 GiB for the anchor [S1].
*   **Quality:** At RP2 on the Bartender dataset, the anchor produces a bitstream of 9.60 MB, compared to 7.44 MB for the proposed VQ method [S1]. The anchor provides consistent color characteristics among adjacent Gaussians due to HEVC spatial prediction [S1].

## Background
The Video-based GSC Anchor (TMCV) was proposed as the implementation of the video-based GSC anchor for 3DGS compression [S1]. It serves as a reference for evaluating and validating proposed technologies in the ongoing GSC standardization activity [S1]. It combines PLAS-based reordering with 2D video coding (HEVC) [S1].

## Sources
[S1] Hybrid Compression Method for Trained 3D Gaussian Splatting Models Based on VQ and HEVC., https://doi.org/10.3390/s26134125, fetched 2026-10-27
