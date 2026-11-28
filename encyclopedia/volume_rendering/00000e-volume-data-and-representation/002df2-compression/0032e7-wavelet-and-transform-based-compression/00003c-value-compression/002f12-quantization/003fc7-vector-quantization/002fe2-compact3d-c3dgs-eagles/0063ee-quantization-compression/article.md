# Quantization & Compression
Also called: 3DGS Quantization, Parameter Quantization

## Overview
This method compresses trained 3D Gaussian Splatting (3DGS) models by quantizing un-activated parameters and applying image codecs to the resulting grids [S1]. It reduces storage size by applying fixed linear quantization ranges derived from dataset percentiles and JPEG XL for storage [S1].

## How it works
**Inputs:** Trained 3DGS model parameters (positions, scales, rotations, opacities, SH coefficients) [S1].
**Outputs:** Compressed file containing quantized indices and metadata [S1].

**Data Structures:**
1.  **Clipped/Normalized Attributes:**
    *   RGB (DC SH): Clipped to $[-2, 4]$, normalized to $[0, 1]$ [S1].
    *   Opacity: Clipped to $[-6, 12]$, normalized to $[0, 1]$ [S1].
    *   Non-DC SH: Clipped to $[-1, 1]$, normalized to $[0, 1]$ [S1].
    *   Rotations: Clipped to $[-1, 2]$, normalized to $[0, 1]$ [S1].
    *   Scales: Clipped (range not specified in sources, but activation is exponential), not normalized to $[0, 1]$ for quantization purposes [S1].
    *   Coordinates: Retained in logarithmic space contraction (see [Compact3D / C3DGS / EAGLES](6139-compact3d-c3dgs-eagles.md)) [S1].
2.  **Quantized Values:**
    *   $q_{\text{coords}} = 2^{14}$ levels [S1].
    *   $q_{\text{scale}} = q_{\text{opacity}} = q_{\text{rotation}} = 2^{6}$ levels [S1].
    *   $q_{\text{SH\_rest}} = 2^{5}$ levels [S1].

**Steps:**
1.  **Activation Retention:** Store parameters in their pre-activation state. Apply sigmoid for opacity, exponential for scale, and logarithmic space contraction for coordinates during rendering [S1].
2.  **Clipping:** Clip RGB to $[-2, 4]$, opacity to $[-6, 12]$, non-DC SH to $[-1, 1]$, rotations to $[-1, 2]$. These ranges cover the 1st to 99th percentile of values across the 360 dataset [S1].
3.  **Normalization:** Normalize all clipped attributes except scaling values to the range $[0, 1]$ [S1].
4.  **Quantization:** Round normalized values to the closest value of a linear range of $q$ total values [S1].
    *   Formula: $v_{\text{quant}} = \lfloor v_{\text{norm}} \times (q - 1) + 0.5 \rfloor$ [G].
    *   Where $v_{\text{norm}} \in [0, 1]$.
    *   $q$ depends on the attribute type as defined above [S1].
5.  **Grid Formation:** Organize quantized RGB values into an 8-bit image grid [S1].
6.  **Compression:** Compress the RGB grid using lossy JPEG XL with 8-bit depth and quality level 100 [S1]. Compress all other attributes (quantized indices for scale, opacity, rotation, non-DC SH, coordinates) using lossless JPEG XL [S1].

**Parameters:**
*   Clipping Ranges: RGB $[-2, 4]$, opacity $[-6, 12]$, non-DC SH $[-1, 1]$, rotations $[-1, 2]$ [S1].
*   Quantization Levels: $q_{\text{coords}}=16384$, $q_{\text{scale}}=q_{\text{opacity}}=q_{\text{rotation}}=64$, $q_{\text{SH\_rest}}=32$ [S1].
*   JPEG XL Quality: 100 (lossy for RGB) [S1].

**Edge Cases:**
*   Values outside the 1st–99th percentile are clipped [S1].
*   Quantization uses fixed bit depths: $2^{14}$ for coordinates, $2^6$ for scale/opacity/rotation, and $2^5$ for non-DC SH [S1].

## Subtopics

### Metadata Structure for VQ Representation
This variant uses Vector Quantization (VQ) instead of linear quantization [S2]. It employs MiniBatch K-means for codebook training with a single initialization, maximum 30 iterations, mini-batch size 4096, and early stopping on no sufficient decrease over 10 consecutive mini-batches [S2]. Random seeds are initialized to 0 for the first-stage coarse VQ and incremented sequentially for residual sub-vector groups; a separate random seed of 0 is used to determine the order in which reference dimensions were selected during the KNN-based grouping procedure [S2]. The metadata payload is fixed at 0.369045 MB per model, consisting of a Coarse VQ Codebook (1024 × 45 Float32, 0.18432 MB), Residual VQ Codebooks (G × 1024 × (45/G) Float32, 0.18432 MB), Normalization Parameters (2 × 45 Float32, 0.00036 MB), and Grouping Information (45 ordered dimension indices, UInt8, 0.000045 MB) [S2]. The codebook size is fixed to 1024, requiring 10 bits per index [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Storage size is reduced significantly compared to raw 3DGS [S1]. The fixed metadata overhead for the VQ variant is 0.369045 MB [S2].
*   **Performance:** Uses off-the-shelf JPEG XL codecs [S1].
*   **Strengths:** Simple implementation using standard image codecs; effective compression for RGB grids [S1].
*   **Limitations:** Fixed clipping ranges are derived from the 360 dataset percentiles [S1].
*   **When to use:** When storage size is critical and standard image codecs are acceptable [S1].
*   **When not to use:** Not given in the sources.

## Background
Originated in "Compact 3D Scene Representation via Self-Organizing Gaussian Grids" by Morgenstern et al. [S1]. It treats the RGB grid as an 8-bit image for JPEG XL compression [S1]. The VQ variant was proposed in "Hybrid Compression Method for Trained 3D Gaussian Splatting Models Based on VQ and HEVC" [S2].

## Sources
[S1] Compact 3D Scene Representation via Self-Organizing Gaussian Grids, https://doi.org/10.1007/978-3-031-73013-9_2, fetched 2026-10-26
[S2] Hybrid Compression Method for Trained 3D Gaussian Splatting Models Based on VQ and HEVC., https://doi.org/10.3390/s26134125, fetched 2026-10-27
