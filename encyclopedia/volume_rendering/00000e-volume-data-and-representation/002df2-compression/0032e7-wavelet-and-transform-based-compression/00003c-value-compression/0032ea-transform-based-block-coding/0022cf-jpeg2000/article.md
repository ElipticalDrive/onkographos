# JPEG2000
Also called: Individual layer coding, Layered Image Coding Comparison

## Overview
JPEG2000 is a wavelet-based compression standard used as a 2D baseline method for volume data, where each 2D layer of a layered image is coded separately using JPEG2000 Part-1 [S1]. It serves as a comparative benchmark against 3D volumetric schemes to evaluate the impact of inter-layer correlation [S1]. Unlike its sibling [TSDF Block Transform Coding](2548-tsdf-block-transform-coding.md), which focuses on specific band structures, JPEG2000 provides a general-purpose framework with features such as lossy-to-lossless performance, scalability, and region of interest (ROI) support [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D image layer (or a sequence of 2D layers extracted from a volume).
*   **Output:** A compressed bitstream conforming to JPEG2000 Part-1.

**Data Structures:**
The method relies on the Discrete Wavelet Transform (DWT) implemented via lifting steps [S2].

**Steps:**
1.  **Preprocessing:** Shift the data range to non-negative values to prepare for integer wavelet transforms [G].
2.  **Wavelet Transform:** Apply the 2D DWT to the layer using lifting steps [S2]. Spatial supports like (060) and (070) can lead to the highest compression ratios, with conditioning terms impacting the minimum rate by approximately 1.2% [S2].
3.  **Quantization:** Quantize the wavelet coefficients to reduce precision [G].
4.  **Entropy Coding:** Encode the quantized coefficients into a bitstream [G].
5.  **Bitstream Assembly:** Assemble the encoded data into the JPEG2000 codestream [G].

**Parameters:**
*   **Wavelet Filter:** Lifting scheme implementation [S2].
*   **Mode:** Progressive (PROG) mode is referenced in performance comparisons [S2].
*   **Bitrate:** Measured in total bits rather than bits per view (bpv) [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** When applied to layered volumes, individual layer coding lacks model information regarding continuity between layers, causing dark colors to appear around layer boundaries in rendered results [S1].
*   **Correlation Assumption:** The method treats layers independently. If inter-layer correlation is significant, this method underperforms compared to 3D schemes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jpeg2000_compress_layer(layer_2d_image, bit_depth):
    # Step 1: Preprocessing [G]
    # Shift data to non-negative range for integer wavelet transforms
    shifted_data = layer_2d_image + 2**(bit_depth - 1)
    
    # Step 2: Discrete Wavelet Transform via lifting steps [S2]
    # Apply 2D DWT using standard lifting scheme filters (e.g., 9-7 or 5-3)
    wavelet_coeffs = discrete_wavelet_transform_lifting(shifted_data)
    
    # Step 3: Quantization [G]
    # Scale and round coefficients based on target bit-rate or lossless setting
    quantized_coeffs = quantize_coefficients(wavelet_coeffs)
    
    # Step 4: Entropy Coding [G]
    # Encode using MQ-coder or arithmetic coding as per Part-1
    bitstream = entropy_encode_coefficients(quantized_coeffs)
    
    return bitstream
```

## Subtopics

### [Volumetric Code-Block EBCOT](7588-volumetric-code-block-ebcot.md)
A coding technique using 3D code-blocks (depth > 1) for volumetric data. It is supported by JP3D but not by JPEG 2000 Part 1 or 2, which only support flat code-blocks (depth=1). Choose it when inter-layer correlation is high and 3D processing is feasible, as it avoids the boundary artifacts of 2D layer coding. See [Volumetric Code-Block EBCOT](7588-volumetric-code-block-ebcot.md).

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the Head sequence (7 layers), a 3D scheme improved reconstructed image quality by up to 5 dB compared to individual layered coding (JPEG2000) [S1].
*   **Scaling:** Supports scalability by quality and resolution [S2].
*   **Strengths:** Improved compression efficiency, lossy-to-lossless performance, scalability, and region of interest (ROI) functionalities compared to standard JPEG [S2].
*   **Limitations:** Lacks inter-layer model information, causing dark colors at layer boundaries in rendered results [S1]. Less effective when inter-layer correlation is significant compared to 3D methods [S1].
*   **When to use:** As a baseline for 2D systems [S2].
*   **When not:** When 3D coding is feasible and inter-layer correlation is significant, as 3D schemes improve quality by up to 5 dB over this method [S1].

## Background
JPEG2000 is a coding standard for still images designed to overcome limitations of JPEG [S2]. It serves as a benchmark for 2-D systems [S2]. The standard uses wavelet-based coding implemented by lifting steps [S2]. It is widely used for its feature set including scalability and region of interest (ROI) support [S2].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
