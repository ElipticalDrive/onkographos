# Residual Coding for Volumetric Compression

Also called: Residual Coding

## Overview
Residual Coding is a technique that improves the compression performance (reconstructed image quality) of volumetric codecs by encoding the difference between the original image and the direct reconstructed image from the volume without quantization or coding [S1]. It does not contribute to view synthesis quality and may deteriorate it [S1]. Unlike standard video codecs where residuals depend on the quantized output, these residuals are derived from the unquantized volume reconstruction, making the encoded residuals similar across all bitrates [S1].

## How it works
**Inputs:** Original view images, volumetric data (used for direct reconstruction without quantization/coding).
**Outputs:** Compressed bitstream containing volume data and residual data.

**Data Structures:**
The bitstream consists of two parts:
1.  Volume data (object coding).
2.  Residual data.
The residual data is structured to allow scalable decoding: "significant parts" of the residual bitstream are decoded based on the remaining bit budget after selecting the volume quality [S1].

**Steps:**
1.  **Direct Reconstruction:** For each view, generate a direct reconstructed image from the volume data *without* applying quantization or entropy coding [S1]. Let $I_{orig}$ be the original image and $I_{recon}$ be this direct reconstruction.
2.  **Residual Calculation:** Calculate the residual for each view as the difference:
    $$ R = I_{orig} - I_{recon} $$
    [S1]
3.  **Encoding:** Encode the residuals alongside the volume data. In open-loop operation, the same residuals are coded for all bitrates to preserve scalability [S1].
4.  **Decoding:**
    a. Select the quality of the volume first [S1].
    b. Decode parts of the residual bitstream until the target bitrate is obtained [S1].
    c. Add the decoded residuals to the reconstructed volume data to improve the final image quality [S1].

**Parameters:**
*   **Bitrate thresholds for residual application:** Residuals are employed at bitrates higher than those where view synthesis quality drops below lossless quality by specific thresholds: -0.05 dB, -0.1 dB, or -0.5 dB [S1].
    *   -0.5 dB represents applications where views at original camera positions are significant.
    *   -0.05 dB represents applications where good quality of virtual views is essential (e.g., 3D rendering, look-around, free-view point) [S1].

**Pitfalls:**
*   Residuals do not greatly improve reconstructed image quality at low bitrates but significantly exacerbate view synthesis quality [S1].
*   Highest compression performance cannot be achieved by choosing the low-quality decoded volume because residuals are obtained from the original volume, not directly from reconstructed images at each particular bitrate [S1].
*   View synthesis performance can degrade up to 0.1 dB at high bitrate when residuals are added [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def encode_residual_coding(original_views, volume_data):
    """
    Encodes volumetric data and residuals for scalable compression.
    """
    encoded_volume = encode_volume(volume_data)
    
    residuals = []
    for view in original_views:
        # Direct reconstruction without quantization or coding
        direct_recon = reconstruct_from_volume(volume_data)
        
        # Calculate residual
        residual = view - direct_recon
        residuals.append(residual)
        
    # Encode residuals. In open-loop, same residuals for all bitrates.
    encoded_residuals = encode_residuals(residuals)
    
    return encoded_volume, encoded_residuals

def decode_residual_coding(encoded_volume, encoded_residuals, target_bitrate, volume_quality_threshold):
    """
    Decodes volumetric data and residuals.
    """
    # 1. Select quality of the volume first
    decoded_volume = decode_volume(encoded_volume, volume_quality_threshold)
    
    # 2. Decode parts of residual bitstream until target bitrate is obtained
    # The bit budget for residuals is: target_bitrate - bitrate_used_for_volume
    residual_bit_budget = target_bitrate - get_bitrate(decoded_volume)
    
    decoded_residuals = decode_residuals_partial(encoded_residuals, residual_bit_budget)
    
    # 3. Reconstruct final image
    # Note: The specific addition step is implied by "improving compression performance"
    # and "correct errors at edges".
    final_views = []
    for i, view in enumerate(original_views): # Assuming original_views structure is known or stored
        # Reconstruct direct view from decoded volume
        direct_recon = reconstruct_from_volume(decoded_volume)
        final_view = direct_recon + decoded_residuals[i]
        final_views.append(final_view)
        
    return final_views
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   At 0.42 bpv with a -0.5 dB acceptable quality criterion, the method achieves the same PSNR (34 dB) as higher bitrate methods (0.68 bpv and 0.56 bpv) but with lower bitrate [S1].
    *   View synthesis performance degradation up to 0.1 dB at high bitrate when residuals are added [S1].
    *   Residuals correct errors at edges caused by imperfect volume construction or loss of high frequencies due to bitstream truncation [S1].
*   **Scaling:** Open-loop operation preserves scalability; the same residuals are coded for all bitrates [S1].
*   **Strengths:** Enhances compression performance (reconstructed image quality) at high bitrates; allows flexibility in image size and quality via scalable bitstream [S1].
*   **Limitations:** Deteriorates view synthesis quality; does not improve quality at low bitrates; highest compression performance is limited by the fact that residuals are derived from the original volume, not the specific bitrate's reconstruction [S1].
*   **When to use:** High bitrates where view synthesis quality is acceptable (above the defined thresholds like -0.05 dB, -0.1 dB, or -0.5 dB below lossless) [S1].
*   **When not to use:** Low bitrates or applications where view synthesis quality is critical and cannot tolerate the degradation [S1].

## Background
Originating from research on colour volumetric compression for realistic view synthesis applications, this method was investigated to improve the quality of reconstructed images in volumetric codecs [S1]. It was developed in the context of multi-view video and volumetric rendering, where balancing image quality and view synthesis performance is crucial [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
