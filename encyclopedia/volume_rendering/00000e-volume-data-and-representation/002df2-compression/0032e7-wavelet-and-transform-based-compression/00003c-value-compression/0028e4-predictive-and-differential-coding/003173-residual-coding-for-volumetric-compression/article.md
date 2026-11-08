# Residual Coding for Volumetric Compression

Also called: Residual Coding

## Overview
Residual coding is a technique that improves the compression performance (reconstructed image quality) of a volumetric codec by encoding the difference between the original image and the direct reconstructed image from the volume without quantization or coding [S1]. It sets itself apart from standard predictive coding by deriving residuals from an unquantized reconstruction, ensuring that the encoded residuals remain similar across all bitrates, unlike MPEG video codec residuals which change with quantization [S1]. However, this method does not contribute to view synthesis quality and may deteriorate it [S1].

## How it works
**Inputs:** Original image data, a volumetric representation constructed from reference views, and a target bitrate budget.
**Outputs:** A bitstream containing the encoded volume data and the encoded residuals.

**Data Structures:**
The method operates on two primary streams:
1.  **Volume Stream:** Contains the encoded volumetric data (object coding).
2.  **Residual Stream:** Contains the encoded residuals. This stream is structured to allow scalable decoding, where parts of the bitstream can be consumed based on the remaining bit budget.

**Steps:**
1.  **Volume Construction:** Construct a volume using information from reference views. This volume is used to generate a "direct reconstructed image" without any quantization or entropy coding applied to the volume data itself [S1].
2.  **Residual Calculation:** For each view, calculate the residual $r$ as the difference between the original image $I_{orig}$ and the direct reconstructed image $I_{recon}$:
    $$ r = I_{orig} - I_{recon} $$
    Note that $I_{recon}$ is derived from the unquantized volume [S1].
3.  **Encoding:** Encode the residuals alongside the volume data. In open-loop operation, the same residuals are coded for all bitrates to preserve scalability [S1].
4.  **Decoding:**
    a.  Select the quality of the volume first.
    b.  Decode parts of the residual bitstream until the target bitrate is obtained [S1].

**Parameters:**
*   **Bitrate Thresholds:** Residuals are employed at bitrates higher than those where view synthesis quality drops below lossless quality by specific thresholds: -0.05 dB, -0.1 dB, or -0.5 dB [S1].
    *   -0.5 dB represents applications where views at original camera positions are significant.
    *   -0.05 dB represents applications requiring good quality of virtual views (e.g., 3D rendering, look-around, free-view point) [S1].

**Pitfalls:**
*   **View Synthesis Degradation:** Residuals significantly exacerbate the quality of view synthesis. The quality of view synthesis is degraded up to 0.1 dB at high bitrate when residuals are added [S1].
*   **Low Bitrate Inefficiency:** Residuals do not greatly improve the quality of reconstructed images at low bitrates [S1].
*   **Suboptimal Performance at Low Quality:** The highest compression performance cannot be achieved by choosing a low-quality decoded volume because the residuals are obtained from the original volume, not directly from the reconstructed images at each particular bitrate [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def residual_coding_encoder(original_images, reference_views):
    """
    Encodes volumetric data with residual coding.
    """
    # Step 1: Construct volume without quantization/coding
    volume = construct_volume(reference_views)
    
    residuals = []
    for img in original_images:
        # Step 2: Generate direct reconstructed image from unquantized volume
        # This is the key distinction: no quantization here
        recon_img = reconstruct_from_volume(volume, img.camera_params)
        
        # Step 3: Calculate residual
        residual = img - recon_img
        residuals.append(residual)
        
    # Step 4: Encode volume and residuals
    # Volume is encoded with standard methods
    volume_bitstream = encode_volume(volume)
    
    # Residuals are encoded. In open-loop, same residuals for all bitrates.
    residual_bitstream = encode_residuals(residuals)
    
    return volume_bitstream, residual_bitstream

def residual_coding_decoder(volume_bitstream, residual_bitstream, target_bitrate):
    """
    Decodes volumetric data with residual coding.
    """
    # Step 4a: Decode volume first
    volume = decode_volume(volume_bitstream)
    
    # Step 4b: Decode parts of residual bitstream based on budget
    # The decoder selects quality of volume first, then consumes residual bits
    remaining_budget = target_bitrate - get_volume_bitrate(volume_bitstream)
    
    # Decode significant parts of residuals until budget is exhausted
    residuals = decode_residuals_scalable(residual_bitstream, remaining_budget)
    
    # Reconstruct images
    reconstructed_images = []
    for i, img_params in enumerate(original_image_params):
        recon_img = reconstruct_from_volume(volume, img_params)
        # Add residual to correct errors at edges/high frequencies
        final_img = recon_img + residuals[i]
        reconstructed_images.append(final_img)
        
    return reconstructed_images
```

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** At 0.42 bpv with a -0.5 dB acceptable quality criterion, the method achieves the same PSNR (34 dB) as higher bitrate methods (0.68 bpv and 0.56 bpv) but with lower bitrate [S1].
*   **Scaling:** The method supports scalability because the same residuals are coded for all bitrates. The decoder can flexibly choose volume quality and residual depth [S1].
*   **Strengths:** Enhances compression performance (reconstructed image quality) and can correct errors at edges caused by imperfect volume construction or loss of high frequencies due to bitstream truncation [S1].
*   **Limitations:** Deteriorates view synthesis quality (up to 0.1 dB degradation at high bitrate) [S1]. Does not improve reconstructed image quality significantly at low bitrates [S1].
*   **When to use:** Use when reconstructed image quality at original camera positions is critical and view synthesis quality is less important, or when operating at high bitrates where the view synthesis degradation is acceptable [S1].
*   **When not to use:** Do not use when view synthesis quality is paramount, or when operating at low bitrates where residuals provide little benefit [S1].

## Background
Residual coding for volumetric compression was investigated in the context of improving compression performance for realistic view synthesis applications [S1]. The technique was analyzed to understand its effect on both compression and view synthesis performances, particularly in open-loop operation to preserve scalability [S1].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
