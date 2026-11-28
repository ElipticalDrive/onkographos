# Residual Coding for Volumetric Compression

Also called: Residual Coding

## Overview
Residual coding is a technique that improves the compression performance (reconstructed image quality) of volumetric codecs by encoding the difference between the original image and the direct reconstructed image from the volume without quantization or coding [S1]. It does not contribute to view synthesis quality and may deteriorate it [S1]. Unlike standard video codecs where residuals depend on the quantization level, here the residuals are derived from the unquantized volume reconstruction, making them similar across all bitrates [S1].

## How it works
**Inputs:** Original image data and volumetric representation data [S1].
**Outputs:** Compressed bitstream containing volume data and residual data [S1].

**Data Structures:**
The bitstream is structured to support scalability, containing the volume data (object coding) and the residual data [S1]. The residual data is partitioned into significant parts that can be decoded independently or in sequence to meet a target bitrate [S1].

**Steps:**
1.  **Volume Reconstruction (Unquantized):** Construct the reconstructed image directly from the volume data without applying quantization or entropy coding to the volume coefficients themselves [S1].
2.  **Residual Calculation:** Calculate the residual for each view as the difference between the original image and the direct reconstructed image from the volume (from Step 1) [S1].
    $$ R = I_{\text{original}} - I_{\text{reconstructed\_unquantized}} $$
    Where $R$ is the residual, $I_{\text{original}}$ is the original image, and $I_{\text{reconstructed\_unquantized}}$ is the image reconstructed from the volume without quantization or coding [S1].
3.  **Encoding:** Encode the residuals alongside the volume data [S1]. In open-loop operation, the same residuals are coded for all bitrates to preserve scalability [S1].
4.  **Decoding:** At the decoder, select the quality of the volume first [S1]. Then, decode parts of the residual bitstream until the target bitrate is obtained [S1].

**Parameters:**
*   **Bitrate Thresholds:** Residuals are employed at bitrates higher than those where view synthesis quality drops below lossless quality by specific thresholds: -0.05 dB, -0.1 dB, or -0.5 dB [S1].
    *   -0.5 dB represents applications where views at original camera positions are significant [S1].
    *   -0.05 dB represents applications where good quality of virtual views is essential (e.g., 3D rendering, look-around, free-view point) [S1].

**Pitfalls and Edge Cases:**
*   **View Synthesis Degradation:** Residuals do not greatly improve reconstructed image quality at low bitrates but significantly exacerbate the quality of view synthesis [S1]. View synthesis performance can degrade up to 0.1 dB at high bitrate when residuals are added [S1].
*   **Suboptimal Performance at Low Quality:** The highest compression performance cannot be achieved by choosing the low-quality decoded volume because the residuals are obtained from the original volume, not directly from the reconstructed images at each particular bitrate [S1].
*   **Scalability Constraint:** Because residuals are derived from the unquantized volume, they are similar across all bitrates, unlike MPEG standard video codec residuals [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_residual_coding(original_images, volume_data):
    # Step 1: Direct reconstruction without quantization/coding
    # The evidence states the reconstructed image is derived directly from volume data
    # without quantization or entropy coding.
    reconstructed_unquantized = volume_data  # Direct representation [G]

    # Step 2: Calculate residuals
    residuals = []
    for i in range(len(original_images)):
        residual = original_images[i] - reconstructed_unquantized[i]
        residuals.append(residual)

    # Step 3: Encode residuals
    # In open-loop operation, same residuals are coded for all bitrates
    # to preserve scalability. The residuals are partitioned into significant parts.
    # Partitioning simulates scalable bitstream generation [G]
    residual_bitstream_parts = [residuals[i] for i in range(len(residuals))]

    # Step 4: Combine with volume data
    # The volume data itself is encoded separately (object coding)
    volume_bitstream = volume_data  # Simplified serialization [G]

    bitstream = {
        'volume': volume_bitstream,
        'residual_parts': residual_bitstream_parts
    }
    return bitstream

def decode_residual_coding(bitstream, target_bitrate):
    # Step 1: Decode volume data first
    # The quality of the volume is selected first
    volume_data = bitstream['volume']

    # Step 2: Decode parts of residual bitstream
    # Decode parts of the residual bitstream until the target bitrate is obtained
    # Calculate bitrate used by volume (simplified metric)
    volume_bitrate = len(volume_data) / 1000  # Arbitrary scaling for budget [G]
    residual_budget = target_bitrate - volume_bitrate

    # Select significant parts of residuals based on budget
    # In practice, this selects the most significant coefficients [G]
    selected_residual_parts = bitstream['residual_parts']

    # Step 3: Reconstruct images
    reconstructed_unquantized = volume_data

    reconstructed_images = []
    for i in range(len(reconstructed_unquantized)):
        # Apply selected residual parts to correct errors at edges/high frequencies
        residual_image = selected_residual_parts[i]
        reconstructed_images.append(reconstructed_unquantized[i] + residual_image)

    return reconstructed_images
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources [G].
*   **Memory Complexity:** Not given in the sources [G].
*   **Measured Performance:**
    *   At 0.42 bpv with a -0.5 dB acceptable quality criterion, the method achieves the same PSNR (34 dB) as higher bitrate methods (0.68 bpv and 0.56 bpv) but with lower bitrate [S1].
    *   View synthesis performance degradation is up to 0.1 dB at high bitrate when residuals are added [S1].
    *   Residuals correct errors at edges which come from imperfect volume construction or loss of high frequencies due to bitstream truncation [S1].
*   **Strengths:** Enhances compression performance (reconstructed image quality) and allows flexibility in image size and quality through scalable bitstream partitioning [S1].
*   **Limitations:** Deteriorates view synthesis quality, especially at low bitrates [S1]. Highest compression performance is not achieved when choosing low-quality decoded volumes because residuals are fixed to the original volume [S1].
*   **When to use:** High bitrates where view synthesis quality is acceptable (above the -0.05, -0.1, or -0.5 dB thresholds) and reconstructed image quality is prioritized over view synthesis [S1].
*   **When not to use:** Low bitrates where view synthesis quality is critical and cannot tolerate the degradation caused by residuals [S1].

## Background
Residual coding for volumetric compression was investigated in the context of improving compression performance for realistic view synthesis applications [S1]. The method was proposed to address the limitations of standard volumetric coding where view synthesis quality might degrade at certain bitrates, while acknowledging that residuals do not help view synthesis [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
