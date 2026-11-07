# Residual Coding for Volumetric Compression

Also called: Residual Coding

## Overview
Residual coding is a technique used in volumetric compression to improve the quality of reconstructed images by encoding the difference between the original view and the direct reconstruction from the volume data without quantization or coding [S1]. Unlike standard predictive coding where residuals depend on the quantized state, this method generates residuals from the unquantized volume, making them consistent across all bitrates [S1]. It is distinct from [Delta Encoding](1096-delta-encoding.md) because it operates on the full volumetric reconstruction rather than simple temporal or spatial deltas, and from [Residual Updates in Transform Domain](5256-residual-updates-in-transform-domain.md) as it applies to the spatial domain image reconstruction directly [S1]. The primary trade-off is that while it enhances image fidelity (PSNR), it can degrade view synthesis quality at high bitrates [S1].

## How it works
**Inputs:**
*   Original view images $I_{orig}$.
*   Volume data $V$ used for view synthesis.
*   Target bitrate budget $B_{target}$.

**Outputs:**
*   Compressed volume data (object coding).
*   Compressed residual bitstream.

**Data Structures:**
*   **Residual Image $R$:** A 2D array of pixel values, same dimensions as the view.
    *   Layout: $R[x, y] = I_{orig}[x, y] - I_{recon\_raw}[x, y]$.
    *   Type: Floating-point or high-precision integer to preserve high-frequency details [S1].
*   **Bitstream:** Contains the volume data followed by the entropy-coded residuals. The residuals are scalable, allowing partial decoding [S1].

**Steps:**

1.  **Direct Reconstruction (Unquantized):**
    Generate a reference reconstruction $I_{recon\_raw}$ from the volume data $V$ without applying quantization or entropy coding truncation. This represents the "ideal" reconstruction from the volume geometry and unquantized coefficients [S1].
    $$ I_{recon\_raw} = \text{Reconstruct}(V) $$

2.  **Residual Calculation:**
    Compute the residual image $R$ for each view by subtracting the direct reconstruction from the original image.
    $$ R[x, y] = I_{orig}[x, y] - I_{recon\_raw}[x, y] $$
    This residual captures high-frequency details and errors from imperfect volume construction or high-frequency loss [S1].

3.  **Encoding:**
    *   Encode the volume data $V$ using the primary compression scheme (object coding).
    *   Encode the residual image $R$ using entropy coding.
    *   **Scalability Constraint:** In open-loop operation, the *same* residual bitstream is generated for all bitrates. This is because $R$ is derived from $I_{recon\_raw}$, which is independent of the quantization level applied to $V$ [S1].

4.  **Decoding and Bitrate Selection:**
    *   The decoder first selects the quality level of the volume data, consuming a base bitrate $B_{vol}$ [S1].
    *   If the target bitrate $B_{target} > B_{vol}$, the decoder reads additional bits from the residual bitstream to improve image quality [S1].
    *   The amount of residual data decoded is determined by the remaining budget: $B_{res} = B_{target} - B_{vol}$ [S1].

5.  **Reconstruction:**
    *   Decode the volume data to get $V_{dec}$.
    *   Reconstruct the base image $I_{base}$ from $V_{dec}$.
    *   Decode the selected portion of the residual bitstream to get $R_{dec}$.
    *   Add the residual to the base image:
        $$ I_{final}[x, y] = I_{base}[x, y] + R_{dec}[x, y] $$

**Parameters:**
*   **Quality Thresholds:** Residuals are typically employed only when the view synthesis quality drops below specific thresholds relative to lossless quality. Common thresholds are -0.05 dB, -0.1 dB, or -0.5 dB [S1].
    *   -0.05 dB: For applications requiring high-quality virtual views (e.g., 3D rendering, look-around) [S1].
    *   -0.5 dB: For applications where original camera positions are significant [S1].
*   **Typical Performance:** At 0.42 bits per voxel (bpv) with a -0.5 dB acceptable quality criterion, the method achieves a PSNR of 34 dB, comparable to methods using 0.56 bpv and 0.68 bpv [S1].

**Edge Cases and Pitfalls:**
*   **View Synthesis Degradation:** Adding residuals can degrade view synthesis quality by up to 0.1 dB at high bitrates [S1]. This occurs because the residuals are optimized for image PSNR, not for the geometric consistency required for view synthesis [S1].
*   **Low Bitrate Inefficiency:** Residuals do not significantly improve reconstructed image quality at low bitrates but still consume bits that could be used for volume data, potentially exacerbating view synthesis quality issues [S1].
*   **Suboptimal Global Performance:** The highest compression performance is not achieved by choosing a low-quality decoded volume and adding residuals. This is because the residuals are fixed based on the original volume, not adapted to the specific quantization errors of the low-quality volume reconstruction [S1].
*   **Scalability Limitation:** Since residuals are derived from the unquantized volume, they are similar across all bitrates, unlike MPEG video codecs where residuals change with quantization [S1].

## Operating characteristics
*   **Time Complexity:** The residual calculation requires one additional reconstruction pass per view without quantization, adding overhead to the encoder. Decoding requires standard volume decoding plus residual decoding, which is linear in the number of bits decoded [S1].
*   **Memory Complexity:** Requires storing the residual bitstream in addition to the volume data. The residual size depends on the entropy of the difference between the original and the unquantized reconstruction [S1].
*   **Performance:** Improves PSNR of reconstructed images, particularly at edges, by correcting errors from imperfect volume construction or bitstream truncation [S1]. However, it may reduce view synthesis quality by up to 0.1 dB at high bitrates [S1].
*   **Strengths:** Enhances image fidelity without requiring complex adaptive residual generation; maintains scalability in open-loop operation [S1].
*   **Limitations:** Can degrade view synthesis quality; not optimal for low bitrates; residuals are static across bitrates [S1].
*   **When to Use:** High-bitrate scenarios where image fidelity is prioritized over view synthesis quality, or when the view synthesis quality is already within acceptable thresholds (e.g., > -0.5 dB degradation) [S1].
*   **When Not to Use:** Low-bitrate scenarios, or applications where view synthesis quality is critical and cannot tolerate the potential 0.1 dB degradation [S1].

## Background
Residual coding for volumetric compression was investigated in the context of color volumetric compression for realistic view synthesis applications [S1]. The method was proposed to address the limitations of standard volumetric codecs where high-frequency details are lost due to quantization or bitstream truncation [S1]. It leverages the fact that the volume reconstruction without quantization provides a good baseline, and the residual captures the fine details that the volume representation alone cannot perfectly convey [S1]. The approach was tested using volumes constructed from five reference views and evaluated on metrics such as PSNR and view synthesis quality [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
