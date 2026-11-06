# JPEG2000
Also called: Individual layer coding, Layered Image Coding Comparison

## Overview
JPEG2000 is a wavelet-based image compression standard used as a baseline 2D coding method for volume data, where each 2D layer of a layered image is coded separately using JPEG2000 Part-1 [S1]. It serves as a comparative benchmark against 3D volumetric schemes to evaluate the impact of inter-layer correlation [S1]. Unlike its sibling [Triadic frequency band decomposition](828-triadic-frequency-band-decomposition.md) which focuses on specific band structures, JPEG2000 provides a general-purpose framework with features such as lossy-to-lossless performance, scalability, and region of interest (ROI) support [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D image layer (or a sequence of 2D layers extracted from a volume).
*   **Output:** A compressed bitstream conforming to JPEG2000 Part-1.

**Data Structures:**
The method relies on the Discrete Wavelet Transform (DWT) implemented via lifting steps [S2]. The specific data structures for the bitstream (e.g., packet organization, codestream syntax) are defined by the JPEG2000 Part-1 standard [S2].

**Steps:**
1.  **Preprocessing:** The input 2D layer is prepared for compression. [G] Standard JPEG2000 preprocessing involves shifting the data range (e.g., adding $2^{bit-depth-1}$ for signed data) to make it non-negative before integer wavelet transforms.
2.  **Wavelet Transform:** Apply the DWT to the 2D layer using lifting steps [S2]. The sources note that spatial supports like (060) and (070) can lead to high compression ratios, though the impact of conditioning terms is small (~1.2% of minimum rate) [S2].
3.  **Quantization:** Quantize the wavelet coefficients. JPEG2000 supports both lossless and lossy quantization [S2].
4.  **Entropy Coding:** Encode the quantized coefficients using the JPEG2000 entropy coding engine (typically MQ-coder) [S2].
5.  **Bitstream Assembly:** Assemble the compressed data into the JPEG2000 codestream [S2].

**Parameters:**
*   **Wavelet Filter:** Lifting scheme implementation [S2].
*   **Mode:** Progressive (PROG) mode is referenced in comparisons [S2].
*   **Bitrate:** Measured in total bits or bits per view (bpv) [S1].

**Edge Cases and Pitfalls:**
*   **Boundary Artifacts:** When applied to layered volumes, individual layer coding lacks model information regarding the continuity between layers. This causes dark colors to appear around layer boundaries in the rendered results [S1].
*   **Correlation Assumption:** The method assumes layers are independent. If inter-layer correlation is high, this method underperforms compared to 3D schemes [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def jpeg2000_compress_layer(layer_2d_image):
    """
    Compress a single 2D layer using JPEG2000 Part-1 logic.
    """
    # Step 1: Preprocessing [G]
    # Shift data to non-negative range if required by the specific integer wavelet implementation
    shifted_data = layer_2d_image + 2**(bit_depth - 1)

    # Step 2: Discrete Wavelet Transform via lifting steps [S2]
    # Apply 2D DWT. The specific filter coefficients (e.g., 9-7 or 5-3) 
    # are defined by the JPEG2000 standard.
    wavelet_coeffs = apply_dwt_lifting(shifted_data)

    # Step 3: Quantization [S2]
    # Quantize coefficients based on desired bit-rate or lossless setting
    quantized_coeffs = quantize(wavelet_coeffs, quantization_step)

    # Step 4: Entropy Coding [S2]
    # Encode using MQ-coder or arithmetic coding as per Part-1
    bitstream = entropy_encode(quantized_coeffs)

    return bitstream
```

## Subtopics

### Individual layer coding
A specific application of JPEG2000 where each 2D slice of a volume is compressed independently. It is chosen when 3D processing is too expensive or when inter-layer correlation is low. It suffers from boundary artifacts due to lack of inter-layer model information [S1]. See [JPEG2000 (method)](52-jpeg2000.md).

### Layered Image Coding Comparison
A comparative evaluation framework using JPEG2000 on layers as a baseline against 3D volumetric coding. It highlights that quality improvements from 3D schemes are smaller for layered images than for full volumes because inter-layer correlation is lower than intra-volume correlation [S1]. See [JPEG2000 (method)](52-jpeg2000.md).

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** In tests on the Head sequence (7 layers), the proposed 3D scheme improved reconstructed image quality by up to 5 dB compared to individual layered coding (JPEG2000) [S1].
*   **Scaling:** JPEG2000 supports scalability by quality and resolution [S2].
*   **Strengths:** Improved compression efficiency, lossy-to-lossless performance, scalability, and ROI functionalities compared to standard JPEG [S2].
*   **Limitations:** Lacks inter-layer model information, leading to visual artifacts (dark colors) at layer boundaries in volume rendering [S1]. Less effective when inter-layer correlation is significant compared to 3D methods [S1].
*   **When to use:** As a baseline for 2D systems or when layers are effectively independent [S2].
*   **When not:** When high inter-layer correlation exists and 3D coding is feasible, as 3D schemes offer better quality (up to 5 dB gain) [S1].

## Background
JPEG2000 is a coding standard for still images designed to overcome limitations of JPEG [S2]. It was developed as a benchmark for 2-D systems in medical data encoding contexts [S2]. The standard uses wavelet-based coding implemented by lifting steps [S2]. It is widely used for its feature set including scalability and ROI [S2].

## Sources
[S1] Colour volumetric compression for realistic view synthesis applications, https://doi.org/10.1007/s11042-010-0484-4, fetched 2026-10-26
[S2] Three-dimensional encoding/two-dimensional decoding of medical data, https://doi.org/10.1109/tmi.2003.809689, fetched 2026-10-26
