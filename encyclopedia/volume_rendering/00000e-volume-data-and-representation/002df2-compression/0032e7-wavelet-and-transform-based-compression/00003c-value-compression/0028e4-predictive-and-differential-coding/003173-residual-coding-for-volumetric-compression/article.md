# Residual Coding for Volumetric Compression

Also called: Residual Coding

## Overview
Residual coding is a technique that improves the compression performance (reconstructed image quality) of volumetric codecs by encoding the difference between the original image and the direct reconstructed image from the volume without quantization or coding [S1]. It does not contribute to view synthesis quality and may deteriorate it [S1]. Unlike standard video codecs where residuals depend on the quantization step, these residuals are derived from an unquantized reconstruction, making them consistent across bitrates [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Original view images, a reconstructed volume model (derived from reference views), and a target bitrate budget.
*   **Output:** Compressed bitstream containing the volume data and residual data.

**Data Structures:**
*   **Residual Map:** A 2D array of values $r_{i,j}$ corresponding to each pixel $(i,j)$ in the view. The values are computed as the difference between the original pixel intensity and the intensity projected from the volume.
*   **Bitstream:** Contains the encoded volume data followed by the encoded residual data. The residual data is structured to allow partial decoding [S1].

**Steps:**
1.  **Volume Reconstruction (Unquantized):** Construct the view from the volume data without applying quantization or entropy coding to the volume coefficients themselves. This yields a "direct reconstructed image" $I_{rec}$.
2.  **Residual Calculation:** For each pixel $(i,j)$ in the view, calculate the residual $r_{i,j}$ as the difference between the original image $I_{orig}$ and the direct reconstructed image $I_{rec}$:
    $$ r_{i,j} = I_{orig}(i,j) - I_{rec}(i,j) $$
    This step is performed once using the full-precision volume representation [S1].
3.  **Residual Encoding:** Encode the residuals alongside the volume data. In open-loop operation, the same residuals are coded for all bitrates to preserve scalability [S1]. The residuals are typically entropy-coded (e.g., Huffman or Arithmetic coding) based on their magnitude, allowing significant parts to be decoded first [S1].
4.  **Decoding and Reconstruction:**
    *   The decoder first selects the quality of the volume (bitrate allocation for the volume data).
    *   The volume is decoded to form $I_{rec}$.
    *   The decoder then selects parts of the residual bitstream until the target bitrate is obtained.
    *   The final reconstructed image is obtained by adding the decoded residuals to the volume reconstruction:
        $$ I_{final}(i,j) = I_{rec}(i,j) + r_{dec}(i,j) $$

**Parameters:**
*   **Bitrate Thresholds:** Residuals are employed at bitrates higher than those where view synthesis quality drops below lossless quality by specific thresholds: -0.05 dB, -0.1 dB, or -0.5 dB [S1].
    *   -0.05 dB: For applications demanding high-quality virtual views (e.g., 3D rendering, look-around) [S1].
    *   -0.5 dB: For applications where views at original camera positions are significant [S1].
*   **Typical Values:** At 0.42 bits per voxel (bpv) with a -0.5 dB acceptable quality criterion, the method achieves the same PSNR (34 dB) as higher bitrate methods (0.68 bpv and 0.56 bpv) but with lower bitrate [S1].

**Edge Cases and Pitfalls:**
*   **View Synthesis Degradation:** Residuals do not greatly improve reconstructed image quality at low bitrates but significantly exacerbate the quality of view synthesis [S1]. View synthesis performance can degrade up to 0.1 dB at high bitrate when residuals are added [S1].
*   **Suboptimal Performance at Low Bitrates:** The highest compression performance cannot be achieved by choosing the low-quality decoded volume because the residuals are obtained from the original volume, not directly from the reconstructed images at each particular bitrate [S1]. This mismatch means residuals may not optimally correct errors introduced by heavy quantization of the volume data [S1].
*   **Scalability Trade-off:** While open-loop operation preserves scalability, it prevents the residuals from adapting to the specific quantization noise of the selected volume quality level [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def residual_coding_encoder(original_views, reference_views, target_bitrates):
    """
    Encodes volumetric data with residual coding.
    """
    # 1. Construct volume from reference views
    volume_model = construct_volume(reference_views)
    
    # 2. Calculate residuals for each view
    residuals = []
    for view in original_views:
        # Reconstruct view from volume WITHOUT quantization/coding
        reconstructed_view = project_volume_to_view(volume_model, view.camera_params)
        
        # Calculate residual: Original - Unquantized Reconstruction
        residual_map = view.image_data - reconstructed_view
        residuals.append(residual_map)
        
    # 3. Encode volume and residuals
    # Volume is encoded with scalable layers
    volume_bitstream = encode_volume_scalable(volume_model)
    
    # Residuals are encoded once, ordered by significance for scalability
    residual_bitstream = encode_residuals_scalable(residuals)
    
    return volume_bitstream, residual_bitstream

def residual_coding_decoder(volume_bitstream, residual_bitstream, target_bitrate):
    """
    Decodes volumetric data with residual coding.
    """
    # 1. Decode volume to desired quality
    # Select volume bits based on target bitrate allocation
    volume_model = decode_volume_scalable(volume_bitstream, target_bitrate)
    
    # 2. Reconstruct view from volume (unquantized projection)
    reconstructed_view = project_volume_to_view(volume_model, camera_params)
    
    # 3. Decode residual bits
    # Calculate remaining bit budget for residuals
    remaining_bitrate = target_bitrate - volume_bitrate_used
    residual_map = decode_residuals_scalable(residual_bitstream, remaining_bitrate)
    
    # 4. Final reconstruction
    final_view = reconstructed_view + residual_map
    
    return final_view
```

## Operating characteristics
*   **Time Complexity:** The encoding requires an additional projection step to generate the unquantized reconstruction for residual calculation, adding $O(N)$ overhead per view where $N$ is the number of pixels. Decoding requires the same projection step plus partial decoding of the residual stream.
*   **Memory Complexity:** Requires storing the full-precision volume model temporarily to generate residuals, and storing the residual maps. The residual bitstream adds to the total storage size.
*   **Measured Performance:**
    *   Improves PSNR of reconstructed images, particularly at edges, by correcting errors from imperfect volume construction or loss of high frequencies [S1].
    *   At 0.42 bpv, achieves 34 dB PSNR, matching higher bitrate methods (0.56-0.68 bpv) without residuals [S1].
    *   View synthesis quality may degrade by up to 0.1 dB at high bitrates [S1].
*   **Strengths:**
    *   Enhances subjective image quality (sharpness, edge fidelity) at lower bitrates.
    *   Maintains scalability in open-loop operation.
*   **Limitations:**
    *   Degrades view synthesis quality.
    *   Suboptimal when low-quality volume reconstructions are selected, as residuals are not adapted to the specific quantization error of that level.
    *   Not suitable for applications where view synthesis quality is the primary metric.

## Background
Residual coding for volumetric compression was investigated in the context of free-viewpoint video and light field compression, specifically for realistic view synthesis applications [S1]. The method was proposed to address the trade-off between image reconstruction quality and view synthesis quality in volumetric codecs [S1]. It differs from MPEG video residuals because it is derived from an unquantized volume reconstruction, ensuring consistency across bitrates [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
