# Residual Coding for Volumetric Compression

Also called: Residual Coding

## Overview
Residual coding is a technique that improves the compression performance (reconstructed image quality) of a volumetric codec by encoding the difference between the original image and the direct reconstructed image from the volume without quantization or coding [S1]. Unlike standard predictive coding which aims to reduce view synthesis artifacts, this method does not contribute to view synthesis quality and may deteriorate it [S1]. It sets itself apart from siblings like [Delta Encoding](1096-delta-encoding.md) by operating on the difference between the *original* source and the *unquantized* volume reconstruction, rather than between adjacent samples or frames, ensuring residuals remain consistent across bitrates [S1].

## How it works
**Inputs:** Original view images, volumetric data structure used for view synthesis, bitrate allocation parameters.
**Outputs:** Compressed volume data, compressed residual bitstream.

**Data Structures:**
1.  **Volume Data:** The primary volumetric representation (e.g., point cloud, voxel grid, or implicit surface) used for synthesizing views.
2.  **Residual Bitstream:** A separate bitstream containing the encoded residuals. It is structured to allow partial decoding to meet specific bitrate targets [S1].

**Steps:**
1.  **Direct Reconstruction:** For each view, generate a direct reconstructed image from the volumetric data *without* applying quantization or entropy coding to the volume data itself. This represents the theoretical maximum quality of the volume-based synthesis [S1].
2.  **Residual Calculation:** Compute the residual $R$ for each view as the pixel-wise difference between the original image $I_{orig}$ and the direct reconstructed image $I_{recon}$:
    $$ R = I_{orig} - I_{recon} $$
    Because the residuals are derived from the unquantized volume reconstruction, the residuals encoded at all bitrates are similar, unlike residuals in standard MPEG video codecs which change with quantization [S1].
3.  **Encoding:** Encode the residuals alongside the volume data. In open-loop operation, the same residuals are coded for all bitrates to preserve scalability [S1].
4.  **Decoding and Bitrate Selection:**
    *   The decoder first selects the quality of the volume data.
    *   It then decodes parts of the residual bitstream until the target bitrate is obtained [S1].
    *   For example, if the acceptable quality of the object requires 0.53 bpv for the volume, and the total required size is 0.7 bpv, the decoder consumes the remaining 0.17 bpv budget to decode significant parts of the residuals [S1].

**Parameters:**
*   **Bitrate Thresholds:** Residuals are employed at bitrates higher than those where view synthesis quality drops below lossless quality by specific margins: -0.05 dB, -0.1 dB, or -0.5 dB [S1].
    *   -0.5 dB degradation represents applications where views at original camera positions are significant [S1].
    *   -0.05 dB degradation represents applications where good quality of virtual views is essential (e.g., 3D rendering, look-around, free-view point) [S1].

**Pitfalls and Edge Cases:**
*   **View Synthesis Degradation:** Residuals do not greatly improve reconstructed image quality at low bitrates but significantly exacerbate the quality of view synthesis [S1]. At high bitrates, adding residuals degrades view synthesis quality by up to 0.1 dB [S1].
*   **Suboptimal Compression Performance:** The highest compression performance cannot be achieved by choosing a low-quality decoded volume. This is because the residuals are obtained from the original volume, not directly from the reconstructed images at each particular bitrate [S1].
*   **Open-Loop Constraint:** The method relies on open-loop operation to preserve scalability. Using closed-loop reconstruction for residual calculation would tie residuals to specific quantization levels, breaking the bitrate scalability [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   At 0.42 bpv with a -0.5 dB acceptable quality criterion, the method achieves the same PSNR (34 dB) as higher bitrate methods (0.68 bpv and 0.56 bpv) but with lower bitrate [S1].
    *   Residuals correct errors at edges caused by imperfect volume construction or loss of high frequencies due to bitstream truncation, achieving sharper edges at lower bitrates compared to smoother textures at higher bitrates [S1].
*   **Scaling:** The method preserves scalability because the same residuals are coded for all bitrates [S1].
*   **Strengths:** Enhances compression performance (reconstructed image quality) at high bitrates; allows flexibility in balancing image size and quality via partial residual decoding [S1].
*   **Limitations:** Deteriorates view synthesis performance; ineffective at low bitrates for image quality improvement; does not achieve optimal compression when low-quality volumes are selected [S1].
*   **When to use:** High-bitrate scenarios where reconstructed image quality is prioritized over view synthesis quality, or when specific edge fidelity is required [S1].
*   **When not to use:** Low-bitrate scenarios or applications where view synthesis quality is critical (e.g., free-view point navigation) [S1].

## Background
Originating from research into colour volumetric compression for realistic view synthesis applications, this method was introduced to address the trade-off between reconstructed image quality and view synthesis quality [S1]. It was developed in the context of codecs that construct volumes from multiple reference views, investigating how residual coding affects both compression and view synthesis performances [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
