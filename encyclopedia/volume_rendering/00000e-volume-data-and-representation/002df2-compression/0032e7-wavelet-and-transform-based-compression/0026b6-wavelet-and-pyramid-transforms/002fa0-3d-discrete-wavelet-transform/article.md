# 3D Discrete Wavelet Transform
Also called: 3D Wavelet Transformation, ICER-3D, ICER-3D Hyperspectral Image Compression Software
## Overview
The 3D Discrete Wavelet Transform (3D DWT) is a multiresolution decomposition method that extends the 2D wavelet algorithm to volumetric data [S1, unofficial]. It decomposes a 3D volume into frequency subbands using a separable filter bank applied along each axis [S1, unofficial]. It sets itself apart from its siblings by applying a standard, axis-aligned separable filter bank to the full 3D grid, whereas variants like [Directional Adaptive Discrete Wavelet Transform](7554-directional-adaptive-discrete-wavelet-transform.md) adapt the transform direction locally, and [Depth-plane Reduction](6446-depth-plane-reduction.md) physically compacts the data before transformation [G].
## How it works
**Inputs and Outputs:**
- Input: A 3D volume $V$ of dimensions $N_x \times N_y \times N_z$, where each voxel contains a scalar value [S1, unofficial].
- Output: A set of wavelet coefficients $C$ organized in a hierarchical structure (pyramid), consisting of one approximation subband and multiple detail subbands (horizontal, vertical, depth, and combinations thereof) at each decomposition level [S1, unofficial].

**Data Structures:**
The output is stored as a set of 3D arrays [G]. For a single level of decomposition on a volume of size $N \times N \times N$:
- Approximation subband $A$: Size $\lceil N/2 \rceil \times \lceil N/2 \rceil \times \lceil N/2 \rceil$ [G].
- Detail subbands: Seven subbands of the same size as $A$, representing high-frequency components along different axes (e.g., $D_{xxx}, D_{xxy}, \dots$) [G].

**Steps:**
The 3D DWT is a separable transform applied in three passes, one for each spatial dimension [S1, unofficial].

1.  **Pass 1 (X-axis):** Apply the 1D DWT to every row (along the X-axis) of the volume. This splits the volume into low-pass (approximation) and high-pass (detail) components along X [G].
2.  **Pass 2 (Y-axis):** Apply the 1D DWT to every column (along the Y-axis) of the results from Pass 1 [G].
3.  **Pass 3 (Z-axis):** Apply the 1D DWT to every depth line (along the Z-axis) of the results from Pass 2 [G].

This process generates the subbands for one level [G]. To create a multi-resolution pyramid, the approximation subband $A$ from level $j$ is recursively decomposed at level $j+1$ [S1, unofficial]. This recursion continues until the dimension of the approximation subband becomes small enough (e.g., $16 \times 16 \times 16$) [S1, unofficial].

Following wavelet decomposition, mean values are subtracted from spatial planes of spatially low-pass subbands prior to encoding [S2, unofficial]. The resulting data are converted to sign-magnitude form [S2, unofficial]. The spatial-frequency domain is partitioned into rectangular error-containment regions that extend through all wavelength bands [S2, unofficial]. Data in each partition are compressed independently [S2, unofficial]. Compressed information is ordered progressively so that successive reconstructions have successively higher overall fidelity as more data is received [S2, unofficial].

**Formulas:**
Let $h$ be the low-pass filter coefficients and $g$ be the high-pass filter coefficients [G]. For a 1D signal $x[n]$, the decomposition is:
$$ A[n] = \sum_k x[k] h[2n - k] $$
$$ D[n] = \sum_k x[k] g[2n - k] $$
In 3D, the approximation coefficient $A[i,j,k]$ at level $L$ is computed by convolving the input with the tensor product of the low-pass filters $h_x \otimes h_y \otimes h_z$ followed by downsampling by 2 in each dimension [G]. The detail coefficients are computed similarly using combinations of $h$ and $g$ (e.g., $g_x \otimes h_y \otimes h_z$ for the horizontal detail band) [G].

**Parameters:**
- Filter type: Not specified in sources. Common choices include Daubechies, Coiflets, or Haar [G].
- Decomposition depth: Determined by the target minimum approximation size (e.g., $16^3$) [S1, unofficial].
- Context model: DWT coefficients are assigned to four categories, numbered 0-3, in preparation for assigning them to contexts [S2, unofficial]. The context modeler classifies bits into one of 19 contexts based on previously encoded bits [S2, unofficial].

**Edge Cases and Pitfalls:**
- Boundary handling: The sources do not specify boundary extension methods (e.g., symmetric, periodic, zero-padding) [S1, unofficial]. Implementers must choose a method consistent with the chosen filter bank [G].
- Dimensionality: The algorithm assumes the volume dimensions are divisible by $2^L$ for $L$ levels, or requires padding to handle non-power-of-two dimensions [G].
- Precision: Standard floating-point arithmetic is implied. For lossless requirements, see [3D Integer Wavelet Transform](6956-3d-integer-wavelet-transform.md) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dwt_3d(volume, low_pass_filter, high_pass_filter):
    # Pass 1: X-axis
    temp_x = apply_1d_dwt_axis(volume, low_pass_filter, high_pass_filter, axis=0)
    # Pass 2: Y-axis
    temp_xy = apply_1d_dwt_axis(temp_x, low_pass_filter, high_pass_filter, axis=1)
    # Pass 3: Z-axis
    coeffs = apply_1d_dwt_axis(temp_xy, low_pass_filter, high_pass_filter, axis=2)
    return coeffs

def apply_1d_dwt_axis(data, h, g, axis):
    low_conv = convolve_1d(data, h, axis)
    high_conv = convolve_1d(data, g, axis)
    if axis == 0:
        low = low_conv[::2, :, :]
        high = high_conv[::2, :, :]
    elif axis == 1:
        low = low_conv[:, ::2, :]
        high = high_conv[:, ::2, :]
    else:
        low = low_conv[:, :, ::2]
        high = high_conv[:, :, ::2]
    return (low, high)

def convolve_1d(data, filter_coeffs, axis):
    # Standard 1D convolution along the specified axis [G]
    # Boundary handling is not specified in sources [S1, unofficial]
    # Implementation uses valid convolution mode [G]
    # [G] The sources do not specify the exact convolution implementation.
    pass
```
## Subtopics

### [Depth-plane Reduction](6446-depth-plane-reduction.md)
A data rearrangement technique that shifts voxels to fill empty depth planes, reducing the total number of planes required for the 3D DWT. It compacts data to concentrate energy in the baseband. Choose when the volume has significant empty space.

### [3D Integer Wavelet Transform](6956-3d-integer-wavelet-transform.md)
A lossless variant that maps integer inputs to integer coefficients using embedded rounding. Choose when perfect reconstruction with finite precision is required.

### [3-D Medical Image Compression with VOI Coding](7407-3-d-medical-image-compression-with-voi-coding.md)
Uses EBCOT on 3D code-cubes to enable scalable Volume of Interest (VOI) retrieval. Choose for telemedicine applications requiring selective decoding.

### [Directional Adaptive Discrete Wavelet Transform](7554-directional-adaptive-discrete-wavelet-transform.md)
Selects the optimal decomposition direction per block to handle non-axis-aligned features. Choose for smooth, non-axis-aligned volumetric data like CT/MRI.

### [3-D SPIHT](8260-3-d-spiht.md)
An embedded entropy coder treating coefficients as 3D orientation trees. Choose for high compression ratios exploiting 3D spatial redundancy.

### [Optimized 3D Wavelet Transform](16208-optimized-3d-wavelet-transform.md)
Reduces computational cost by processing only a subset of rows in the third dimension. Choose when speed is prioritized over full symmetric decomposition.

## Operating characteristics
- **Time Complexity:** $O(N^3)$ for a volume of size $N^3$, as each voxel is processed a constant number of times per level [G]. Recursive decomposition adds a logarithmic factor in depth [G].
- **Memory Complexity:** $O(N^3)$ for the input and output coefficients [G]. The hierarchical structure requires storing all subbands [G].
- **Performance:** The method is straightforward and computationally efficient compared to adaptive variants [G].
- **Strengths:** Simple implementation, well-understood properties, effective for general-purpose volume compression [G].
- **Limitations:** Fixed axis-aligned decomposition may not align with local geometries in smooth volumetric data [G]. Requires careful boundary handling [G].
- **When to use:** General-purpose volume compression where axis-aligned features are dominant or when a simple baseline is needed [G].
- **When not to use:** When lossless integer arithmetic is required (use [3D Integer Wavelet Transform](6956-3d-integer-wavelet-transform.md)), or when data has strong non-axis-aligned structures (use [Directional Adaptive Discrete Wavelet Transform](7554-directional-adaptive-discrete-wavelet-transform.md)) [G].
## Background
The 3D Discrete Wavelet Transform is a straightforward extension of the 2D wavelet algorithm to 3D data [S1, unofficial]. It has been used in computer graphics and engineering for image and volume compression [S1, unofficial]. The method relies on the separability of the 3D transform into three 1D passes [S1, unofficial].
## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] NASA Technical Reports Server (NTRS) 20100005261: ICER-3D Hyperspectral Image Compression Software, https://archive.org/details/NASA_NTRS_Archive_20100005261, fetched 2026-10-26 (unofficial)
