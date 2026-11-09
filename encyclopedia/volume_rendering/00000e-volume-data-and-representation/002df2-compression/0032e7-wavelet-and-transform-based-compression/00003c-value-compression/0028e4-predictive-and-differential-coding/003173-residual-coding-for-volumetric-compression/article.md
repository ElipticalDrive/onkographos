# Residual Coding for Volumetric Compression

Also called: Residual Coding

## Overview
Residual Coding is a technique that improves the compression performance (reconstructed image quality) of a volumetric codec by encoding the difference between the original image and the direct reconstructed image from the volume without quantization or coding [S1]. It does not contribute to view synthesis quality and may deteriorate it [S1]. Unlike standard video codecs where residuals depend on the quantized output, these residuals are derived from the unquantized volume reconstruction, making them consistent across all bitrates [S1].

## How it works
**Inputs:** Original image data $I_{orig}$, direct reconstructed image data $I_{recon}$ (generated from the volume without quantization or entropy coding).
**Outputs:** Residual bitstream $B_{res}$, Volume bitstream $B_{vol}$.

**Data Structures:**
The residual is a 2D image (or set of images for multi-view) with the same dimensions as the original view. Each pixel value $r$ is an integer difference.

**Steps:**
1.  **Generate Direct Reconstruction:** Construct the image $I_{recon}$ directly from the volumetric representation. This step bypasses quantization and entropy coding to ensure the residual reflects only the geometric/topological error of the volume representation, not the quantization error [S1].
2.  **Calculate Residual:** For each pixel $p$, compute the residual $r_p$:
    $$ r_p = I_{orig}(p) - I_{recon}(p) $$
    Because $I_{recon}$ is derived from the same volume for all bitrates, the residuals $r_p$ are identical regardless of the final bitrate selected [S1].
3.  **Entropy Encode:** Encode the residuals $r_p$ using an entropy coder (e.g., Huffman or Arithmetic coding) [S1]. The residuals are encoded alongside the volume data. In open-loop operation, the same residuals are coded for all bitrates to preserve scalability [S1].
4.  **Bitrate Allocation Strategy:** Residuals are employed only at bitrates higher than those where view synthesis quality drops below a specific lossless threshold. Common thresholds are -0.05 dB, -0.1 dB, or -0.5 dB relative to lossless quality [S1].
5.  **Decoding:**
    a. Select the desired quality level for the volume (decode $B_{vol}$).
    b. Decode parts of the residual bitstream $B_{res}$ until the target total bitrate is obtained [S1].
    c. Reconstruct the final image:
    $$ I_{final}(p) = I_{recon}(p) + r_p $$

**Pitfalls:**
*   **View Synthesis Degradation:** Residuals do not greatly improve reconstructed image quality at low bitrates but significantly exacerbate the quality of view synthesis [S1]. View synthesis performance can degrade up to 0.1 dB at high bitrates when residuals are added [S1].
*   **Suboptimal Low-Bitrate Performance:** The highest compression performance cannot be achieved by choosing a low-quality decoded volume because the residuals are obtained from the original volume, not directly from the reconstructed images at each particular bitrate [S1].
*   **Scalability vs. Efficiency Trade-off:** While coding the same residuals for all bitrates preserves scalability, it prevents optimizing the residual for specific quantization levels, potentially leading to suboptimal PSNR at specific bitrates compared to closed-loop methods [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels for residual calculation and encoding. The overhead is the storage of the residual bitstream.
*   **Memory Complexity:** Requires storing the residual bitstream in addition to the volume data.
*   **Performance:** At 0.42 bpv with a -0.5 dB acceptable quality criterion, the method achieves the same PSNR (34 dB) as higher bitrate methods (0.68 bpv and 0.56 bpv) but with lower bitrate [S1]. However, subjective quality differs: lower bitrates with residuals yield sharper edges, while higher bitrates yield smoother textures [S1].
*   **Strengths:** Enhances reconstructed image quality (PSNR) at high bitrates; corrects errors at edges caused by imperfect volume construction or loss of high frequencies due to bitstream truncation [S1].
*   **Limitations:** Deteriorates view synthesis quality; not suitable for applications where virtual view quality is critical (e.g., free-view point applications) unless the bitrate threshold is set very strictly (-0.05 dB) [S1].

## Background
Residual Coding in this context was investigated in the context of colour volumetric compression for realistic view synthesis applications [S1]. It addresses the limitation of volumetric codecs where the reconstructed texture may lack high-frequency details or contain edge artifacts due to imperfect volume construction or bitstream truncation [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
