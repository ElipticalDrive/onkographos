# Wavelet Transform

Also called: Discrete Wavelet Transform, Fast Orthogonal Wavelet Transform, Fast Wavelet Transform, Mallat's Fast Wavelet Transform

## Overview
The Wavelet Transform is a multiresolution analysis method that decomposes a signal or volume into a hierarchy of approximation and detail coefficients using filter banks [S1, unofficial][S4, S5]. It solves the problem of fixed-resolution analysis inherent in windowed Fourier transforms by providing spatially localized frequency information, enabling efficient sparse representation and compression of volumetric data [S1, unofficial][S4]. Unlike the Laplacian pyramid, it is orientation-selective, and unlike the Discrete Cosine Transform (DCT), it avoids block artifacts by using basis functions with local support in both space and frequency [S1, S2, unofficial]. It builds on [Wavelet-based Multiresolution Rendering](1588-wavelet-based-multiresolution-rendering.md) by providing the specific mathematical decomposition engine.

## How it works
**Inputs and Outputs:**
- Input: A discrete signal or volume $f$ (e.g., voxel grid).
- Output: A set of approximation coefficients $c^L$ at the coarsest level $L$, and detail coefficients $d^1, \dots, d^L$ for each level.

**Data Structures:**
The output is typically stored as a hierarchical tree or a flat array of subbands. For an $N$-dimensional input, the decomposition at level $j$ produces:
- One approximation subband (low-pass in all dimensions).
- Detail subbands corresponding to high-pass filtering in one or more dimensions.
*Note on Dimensionality:* Standard image processing (2D) yields 3 detail subbands per level (LH, HL, HH) plus the LL approximation. However, for volume rendering multiresolution, a specific variant is often used where only the approximation coefficients are recursively decomposed, generating one detail subband per dimension per level [S5]. This reduces computational cost and improves compression compared to the full tensor product decomposition [S5].

**Parameters:**
- Wavelet Filter: Determines the decomposition nature. Common choices include Linear B-spline wavelets [S5]:
  - Low-pass (scaling) filter $h$: $h[n] = \frac{1}{8}[-1, 2, 6, 2, -1]$ (for the detail path in some formulations) or $h[n] = \frac{1}{2}[1, 2, 1]$ (standard averaging) [S5].
  - High-pass (wavelet) filter $g$: $g[n] = \frac{1}{2}[1, -2, 1]$ or $g[n] = \frac{1}{8}[1, 2, -6, 2, 1]$ [S5].
- Decomposition Levels ($L$): Finite number of recursive steps. Stopping condition is typically when the approximation subband reaches a minimum dimension (e.g., $16 \times 16$) [S1, unofficial].
- Thresholding: Coefficients below a certain magnitude may be zeroed out for lossy compression [S4, S5].

**Steps (Decomposition/Analysis):**
1. **1D Decomposition:** For a 1D signal $c^j$, compute approximation $c^{j+1}$ and detail $d^{j+1}$ using convolution with filters $h$ and $g$, followed by downsampling by 2:
   $$c^{j+1}_k = \sum_n h(n - 2k) c^j_n$$
   $$d^{j+1}_k = \sum_n g(n - 2k) c^j_n$$
   [S1, unofficial][S5]
2. **2D/3D Extension:** Apply the 1D transform alternately to each dimension.
   - In 2D: Apply 1D transform to rows, then to columns of the result [S1, unofficial].
   - In 3D: Cycle through x, y, and z directions [S5].
   - *Variant for Volume Rendering:* To maintain the specific multiresolution structure used in compressed rendering, only the approximation coefficients (low-pass in all directions) are passed to the next level. The detail coefficients from each directional pass are stored as the detail subbands for that level [S5].
3. **Recursion:** Repeat steps 1-2 on the approximation coefficients $c^j$ until level $L$ is reached [S1, unofficial][S4].

**Steps (Reconstruction/Synthesis):**
1. **Upsampling:** Precede each coefficient sequence ($c^L, d^L, \dots, d^1$) with upsampling by 2 (inserting zeros between samples) [S5].
2. **Filtering:** Convolve upsampled sequences with reconstruction filters (typically the same $h$ and $g$ for orthogonal wavelets, or specific dual filters for biorthogonal) [S5].
3. **Summation:** Sum the filtered results to reconstruct the signal at the previous level [S5].
4. **Inverse 2D/3D:** Reverse the dimensional passes.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wavelet_decompose(signal, filters_h, filters_g, levels):
    """
    Decomposes a signal using the Fast Wavelet Transform.
    signal: 1D array of coefficients at current level.
    filters_h: Low-pass filter coefficients.
    filters_g: High-pass filter coefficients.
    levels: Number of decomposition levels.
    Returns: List of detail coefficients [d1, d2, ..., dL] and final approximation cL.
    """
    details = []
    current_signal = signal
    
    for level in range(levels):
        # 1D Decomposition step
        # Convolve with low-pass filter h
        approx_convolved = convolve(current_signal, filters_h)
        # Convolve with high-pass filter g
        detail_convolved = convolve(current_signal, filters_g)
        
        # Downsample by 2 (keep every second element, starting from index 0 or 1 depending on filter alignment)
        # Standard dyadic downsampling:
        approx_coeffs = approx_convolved[0::2]
        detail_coeffs = detail_convolved[0::2]
        
        details.append(detail_coeffs)
        current_signal = approx_coeffs
        
    return details, current_signal

def wavelet_reconstruct(details, final_approx, filters_h, filters_g, levels):
    """
    Reconstructs signal from wavelet coefficients.
    details: List of detail coefficients [dL, ..., d1] (reverse order of decomposition)
    final_approx: Approximation coefficients cL
    """
    current_signal = final_approx
    
    # Iterate from coarsest detail to finest
    for level in range(levels - 1, -1, -1):
        detail_coeffs = details[level]
        
        # Upsample by 2 (insert zeros)
        approx_upsampled = upsample(current_signal)
        detail_upsampled = upsample(detail_coeffs)
        
        # Filter
        approx_recon = convolve(approx_upsampled, filters_h)
        detail_recon = convolve(detail_upsampled, filters_g)
        
        # Sum to get previous level
        current_signal = approx_recon + detail_recon
        
    return current_signal

def convolve(signal, kernel):
    # Standard linear convolution
    # Implementation detail: pad signal if necessary to handle boundaries
    result = [0] * (len(signal) + len(kernel) - 1)
    for i, k in enumerate(kernel):
        for j, s in enumerate(signal):
            result[i + j] += k * s
    return result

def upsample(signal):
    # Insert zeros between elements
    result = [0] * (2 * len(signal))
    for i, s in enumerate(signal):
        result[2 * i] = s
    return result
```
*Note: The above pseudocode represents the 1D core. For 2D/3D, it must be applied sequentially to dimensions. For the specific volume rendering variant, only the "approx" path is recursed [S5].*

## Subtopics

### [Haar Wavelet Transform](11152-haar-wavelet-transform.md)
Uses the simplest step-function wavelet. Computationally trivial but less smooth than spline wavelets. Chosen for speed when smoothness is not critical.

## Operating characteristics
- **Time Complexity:** Linear time $O(N)$ for decomposition and reconstruction, where $N$ is the number of data points [S2, unofficial].
- **Memory Complexity:** Requires storage for all coefficients. The total size is equal to the original data size for lossless decomposition (orthonormal basis). Lossy compression reduces effective size via thresholding [S4, S5].
- **Performance:** Achieves high compression ratios (e.g., 64:1) with visually invisible artifacts due to the smooth distribution of quantization error [S2, unofficial].
- **Strengths:** Local support in space and frequency eliminates block artifacts common in DCT (JPEG). Orientation sensitivity in 2D/3D allows better feature representation than Laplacian pyramids [S1, S2, unofficial].
- **Limitations:** Boundary effects can occur if not handled properly. Lossy compression introduces errors that are smoothed by the inverse transform but are present [S2, unofficial].
- **Usage:** Ideal for block-wise hierarchical decomposition (octree) with entropy encoding, block caching, and projective classification in compressed GPU-based direct volume rendering [S4].

## Background
The wavelet transform was introduced for compressed volume rendering by Muraki in 1993 [S4]. It was developed to overcome the fixed resolution limits of windowed Fourier (Gabor) transforms, which cannot analyze signals with features of vastly different sizes optimally [S1, unofficial]. Stephan Mallat formalized the fast orthogonal wavelet transform algorithm, exploiting multiresolution analysis to create an efficient, orientation-selective decomposition superior to the Laplacian pyramid [S1, unofficial]. The method has roots in signal processing and pure mathematics, utilizing dilated and translated versions of a single "mother" wavelet function [S2, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
[S3] Wavelet Based 3D Compression for Very Large Volume Data Supporting Fast Random Access, https://www.brics.dk/RS/99/34/index.html, fetched 2026-11-04
[S4] PDF State-of-the-art in Compressed GPU-Based Direct Volume Rendering, https://www.ifi.uzh.ch/dam/jcr:00000000-1fe0-1a53-ffff-ffff94c9f943/CompressedDVR.pdf, fetched 2026-10-27
[S5] Multiresolution Distance Volumes for Progressive Surface Compression, https://digital.library.unt.edu/ark:/67531/metadc1409942/?q=volume data compression rendering, fetched 2026-10-28
