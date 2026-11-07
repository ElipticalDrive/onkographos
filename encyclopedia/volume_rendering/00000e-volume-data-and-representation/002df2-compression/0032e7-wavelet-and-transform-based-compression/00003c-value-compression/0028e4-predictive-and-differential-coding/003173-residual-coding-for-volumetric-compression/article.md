# Residual Coding for Volumetric Compression

Also called: Residual Coding

## Overview
Residual coding is a technique that improves the compression performance (reconstructed image quality) of volumetric codecs by encoding the difference between the original image and the direct reconstructed image from the volume without quantization or coding [S1]. It is distinct from standard predictive coding because the residuals are derived from an unquantized reconstruction, meaning the residuals encoded at all bitrates are similar, unlike residuals in MPEG standard video codecs [S1]. While it enhances the PSNR of the reconstructed texture, it does not contribute to view synthesis quality and may deteriorate it [S1].

## How it works
**Inputs:**
*   Original image data $I_{orig}$.
*   Directly reconstructed image data $I_{rec}$, generated from the volume representation without any quantization or entropy coding [S1].

**Outputs:**
*   Encoded volume data.
*   Encoded residual data $R$.

**Data Structures:**
*   **Residual Map:** A 2D array (or 3D if applied per-voxel in the volume domain, but the source describes "residual of each view") where each element $r_{i,j} = I_{orig}(i,j) - I_{rec}(i,j)$ [S1].
*   **Bitstream:** Contains the volume bitstream and the residual bitstream. The residual bitstream is structured to allow partial decoding [S1].

**Steps:**
1.  **Reconstruction (Unquantized):** Generate $I_{rec}$ from the volume representation. This step must be lossless relative to the volume parameters to ensure the residual reflects only the reconstruction error, not quantization error [S1].
2.  **Residual Calculation:** Compute the residual for each pixel/view as:
    $$ r = I_{orig} - I_{rec} $$
    [S1]
3.  **Encoding:** Encode the residuals $r$ alongside the volume data. In open-loop operation, the same residuals are coded for all bitrates to preserve scalability [S1]. The residuals are typically entropy-coded (e.g., Huffman or Arithmetic) based on their distribution [S1].
4.  **Decoding Strategy:**
    *   The decoder first selects the quality of the volume (bitrate allocation for the volume data) [S1].
    *   The decoder then decodes parts of the residual bitstream until the target total bitrate is obtained [S1].
    *   The final reconstructed image is:
        $$ I_{final} = I_{rec} + r_{decoded} $$
        [S1]

**Parameters:**
*   **Bitrate Thresholds:** Residuals are employed at bitrates higher than those where view synthesis quality drops below lossless quality by specific margins: -0.05 dB, -0.1 dB, or -0.5 dB [S1].
    *   -0.5 dB: Represents applications where views at original camera positions are significant [S1].
    *   -0.05 dB: Represents applications where good quality of virtual views is essential (e.g., 3D rendering, look-around, free-view point) [S1].

**Edge Cases and Pitfalls:**
*   **View Synthesis Degradation:** Residuals do not greatly improve reconstructed image quality at low bitrates but significantly exacerbate the quality of view synthesis [S1]. View synthesis performance can degrade up to 0.1 dB at high bitrate when residuals are added [S1].
*   **Suboptimal Performance at Low Bitrates:** The highest compression performance cannot be achieved by choosing the low-quality decoded volume because the residuals are obtained from the original volume, not directly from the reconstructed images at each particular bitrate [S1].
*   **Scalability vs. Efficiency:** While open-loop operation preserves scalability (same residuals for all bitrates), it prevents optimal bit allocation because the residuals are static regardless of the volume's quantization level [S1].

## Operating characteristics
*   **Time Complexity:** The method adds a linear pass for residual calculation ($O(N)$) and an additional entropy coding pass for the residuals. Decoding requires parsing the volume stream and then selectively parsing the residual stream.
*   **Memory Complexity:** Requires storing or transmitting the residual data in addition to the volume data. The residual map size is equal to the image size [S1].
*   **Measured Performance:**
    *   At 0.42 bpv with a -0.5 dB acceptable quality criterion, the method achieves the same PSNR (34 dB) as higher bitrate methods (0.68 bpv and 0.56 bpv) but with lower bitrate [S1].
    *   It allows achieving acceptable PSNR values at low bitrates, correcting errors at edges caused by imperfect volume construction or loss of high frequencies due to bitstream truncation [S1].
*   **Strengths:**
    *   Improves compression performance (PSNR) of reconstructed images [S1].
    *   Preserves scalability in open-loop operation [S1].
    *   Corrects edge errors and high-frequency loss [S1].
*   **Limitations:**
    *   Deteriorates view synthesis quality [S1].
    *   Ineffective at low bitrates for view synthesis [S1].
    *   Static residuals prevent optimal bit allocation across different quality levels [S1].
*   **When to Use:** Use when reconstructed image quality (texture fidelity) is prioritized over view synthesis quality, or when operating at high bitrates where the view synthesis degradation is acceptable (e.g., < -0.1 dB) [S1].
*   **When Not to Use:** Do not use when view synthesis quality is critical, especially at low bitrates, or when optimal bitrate scalability across varying quality levels is required [S1].

## Background
This method was investigated in the context of colour volumetric compression for realistic view synthesis applications [S1]. It addresses the limitation that standard volumetric compression may lose high-frequency details or introduce edge errors, which can be corrected by explicitly coding the difference between the original and the reconstructed view [S1]. The approach highlights the trade-off between image reconstruction fidelity and the geometric consistency required for view synthesis [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
