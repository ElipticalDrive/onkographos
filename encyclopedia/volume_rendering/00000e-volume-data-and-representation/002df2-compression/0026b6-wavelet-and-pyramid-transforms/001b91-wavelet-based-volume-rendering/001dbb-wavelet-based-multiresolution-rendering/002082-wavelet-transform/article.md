# Wavelet Transform

Also called: Fast Wavelet Transform, Discrete Wavelet Transform, Morlet Wavelet Transform

## Overview
The Wavelet Transform provides a sparse representation for highly detailed functions by decomposing signals into dilated and translated versions of a unique basis function [S2, unofficial]. It solves the limitation of windowed Fourier transforms, which suffer from fixed spatial and frequency resolution, by allowing variable resolution analysis suitable for signals with features of vastly different scales [S1, unofficial]. Unlike the Discrete Cosine Transform (DCT) used in JPEG, which requires image fragmentation due to global basis support, the Wavelet Transform has local support in both the spatial and frequency domains, eliminating the need for block-based processing [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A signal or image data array $I$ of size $N$.
*   **Output:** A set of wavelet coefficients representing the signal at multiple resolutions.

**Data Structures:**
The transform produces a hierarchical structure of sub-bands [S2, unofficial]. In 2D, a single level of decomposition splits the input into four sub-bands: $LL$ (low-pass in both directions, approximation), $LH$ (low-pass in x, high-pass in y, horizontal details), $HL$ (high-pass in x, low-pass in y, vertical details), and $HH$ (high-pass in both directions, diagonal details) [S2, unofficial]. The $LL$ sub-band is recursively decomposed in subsequent levels, creating a tree structure (e.g., $LLL$, $LLH$, etc.) [S2, unofficial]. The detail bands ($LH$, $HL$, $HH$) are not further decomposed in standard 2D pyramids [S2, unofficial].

**Parameters:**
*   **Compression Ratio:** Lossy compression can achieve ratios in the order of 64:1 with almost invisible artifacts [S2, unofficial].
*   **Wavelet Basis:** The specific choice of the mother wavelet $\psi(x)$ determines the filter coefficients [S2, unofficial]. The sources reference linear spline wavelets and the lifting scheme for construction [S2, unofficial], but do not provide specific filter coefficients for a standard basis (e.g., Daubechies) [G]. Implementers must select a standard orthogonal wavelet filter bank or use the lifting scheme coefficients defined in the chosen library or standard [G].

**Algorithm Steps:**
1.  **Decomposition (Analysis):** The process recursively separates high-frequency details from lower frequencies [S2, unofficial].
    a. Apply low-pass filtering and high-pass filtering to the input signal in the x-direction.
    b. Downsample the results by a factor of 2 in the x-direction.
    c. Apply low-pass filtering and high-pass filtering to the results from step (b) in the y-direction.
    d. Downsample the results by a factor of 2 in the y-direction.
    e. This yields the four sub-bands: $LL$, $LH$, $HL$, $HH$ [S2, unofficial].
    f. If further decomposition is required, apply steps (a) through (e) recursively to the $LL$ sub-band [S2, unofficial].
2.  **Reconstruction (Synthesis):** The inverse process assembles details from the coefficients [S2, unofficial].
    a. Upsample the sub-bands by a factor of 2 in the respective directions.
    b. Apply synthesis low-pass and high-pass filters.
    c. Sum the filtered results to reconstruct the signal at the current resolution.
    d. Recursively reconstruct higher-resolution levels until the original signal size is reached [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def wavelet_transform_2d(signal, levels, low_pass_filter, high_pass_filter):
    """
    Performs a 2D discrete wavelet transform using the separable filter bank method.
    Matches the formulation: c^1_k = sum_n h(n - 2k) c^0_n, d^1_k = sum_n g(n - 2k) c^0_n [S1, unofficial].
    """
    current_data = signal
    coefficients = []

    for level in range(levels):
        # 1D transform along rows (x-direction)
        low_x = [sum(low_pass_filter[n - 2*k] * current_data[r, n] for n in range(len(low_pass_filter) + len(current_data[r]) - 1)) for k in range(len(current_data[r]) // 2)]
        high_x = [sum(high_pass_filter[n - 2*k] * current_data[r, n] for n in range(len(high_pass_filter) + len(current_data[r]) - 1)) for k in range(len(current_data[r]) // 2)]

        # 1D transform along columns (y-direction) on the downsampled row results
        ll = [[sum(low_pass_filter[m - 2*j] * low_x[i][m] for m in range(len(low_pass_filter) + len(low_x[0]) - 1)) for j in range(len(low_x[0]) // 2)] for i in range(len(low_x))]
        lh = [[sum(high_pass_filter[m - 2*j] * low_x[i][m] for m in range(len(high_pass_filter) + len(low_x[0]) - 1)) for j in range(len(low_x[0]) // 2)] for i in range(len(low_x))]
        hl = [[sum(low_pass_filter[m - 2*j] * high_x[i][m] for m in range(len(low_pass_filter) + len(high_x[0]) - 1)) for j in range(len(high_x[0]) // 2)] for i in range(len(high_x))]
        hh = [[sum(high_pass_filter[m - 2*j] * high_x[i][m] for m in range(len(high_pass_filter) + len(high_x[0]) - 1)) for j in range(len(high_x[0]) // 2)] for i in range(len(high_x))]

        coefficients.append({'LL': ll, 'LH': lh, 'HL': hl, 'HH': hh})
        current_data = ll

    return coefficients

def inverse_wavelet_transform_2d(coefficients, original_shape, low_pass_filter, high_pass_filter):
    """
    Reconstructs the signal from wavelet coefficients.
    """
    current_data = coefficients[-1]['LL']
    for level in range(len(coefficients) - 2, -1, -1):
        bands = coefficients[level]
        ll, lh, hl, hh = bands['LL'], bands['LH'], bands['HL'], bands['HH']

        # Upsample by inserting zeros
        def upsample_1d(arr, axis):
            if axis == 1: # rows
                return [[arr[i][j//2] if j % 2 == 0 else 0 for j in range(len(arr[0]) * 2)] for i in range(len(arr))]
            else: # columns
                return [[arr[i//2][j] if i % 2 == 0 else 0 for j in range(len(arr[0]))] for i in range(len(arr) * 2)]

        ll_up = upsample_1d(ll, 1)
        lh_up = upsample_1d(lh, 1)
        hl_up = upsample_1d(hl, 1)
        hh_up = upsample_1d(hh, 1)

        # Synthesis filtering (convolution)
        def synthesize_1d(arr, filt, axis):
            if axis == 1:
                return [[sum(filt[n - 2*k] * arr[i][n] for n in range(len(filt) + len(arr[i]) - 1)) for k in range(len(arr[i]))] for i in range(len(arr))]
            else:
                return [[sum(filt[m - 2*j] * arr[m][j] for m in range(len(filt) + len(arr))) for j in range(len(arr[0]))] for i in range(len(arr))]

        ll_y = synthesize_1d(ll_up, low_pass_filter, 0)
        lh_y = synthesize_1d(lh_up, high_pass_filter, 0)
        hl_y = synthesize_1d(hl_up, low_pass_filter, 0)
        hh_y = synthesize_1d(hh_up, high_pass_filter, 0)

        ll_x = synthesize_1d(ll_y, low_pass_filter, 1)
        lh_x = synthesize_1d(lh_y, high_pass_filter, 1)
        hl_x = synthesize_1d(hl_y, low_pass_filter, 1)
        hh_x = synthesize_1d(hh_y, high_pass_filter, 1)

        current_data = [[ll_x[i][j] + lh_x[i][j] + hl_x[i][j] + hh_x[i][j] for j in range(len(ll_x[0]))] for i in range(len(ll_x))]
    return current_data
```

**Edge Cases and Pitfalls:**
*   **Quantization Error:** In lossy compression, quantization errors are introduced during the transform range and are smoothly distributed across the image domain by the inverse transform, making them often unrecognizable to the human eye [S2, unofficial].
*   **Boundary Conditions:** The sources do not specify boundary handling for finite signals [G]. Implementers must choose a strategy (e.g., periodic, symmetric, zero-padding) which affects the transform's energy compaction properties [G].
*   **Degenerate Input:** The recursive nature requires the data dimensions to be divisible by $2^L$ where $L$ is the number of levels, or appropriate boundary handling must be implemented to manage non-power-of-two sizes [G].

## Operating characteristics
*   **Time Complexity:** The discrete wavelet transform is computed in linear time, $O(N)$, where $N$ is the number of data points [S2, unofficial].
*   **Memory Complexity:** The transform is typically performed in-place or requires storage proportional to the input size for the coefficient tree [G].
*   **Performance:** Lossy compression achieves high compression rates in the order of 64:1 with almost invisible artifacts [S2, unofficial].
*   **Strengths:** Local support in both image domain and frequency space eliminates the need for image fragmentation (unlike DCT) [S2, unofficial]. It provides a sparse representation for highly detailed functions [S2, unofficial].
*   **Limitations:** The main drawback of the related windowed Fourier transform (Gabor) is constant spatial and frequency resolution, which is not acceptable for signals with features of very different sizes; the wavelet transform avoids this [S1, unofficial].

## Background
The wavelet transform was defined by Monet Morlet to avoid the fixed resolution limit of windowed Fourier transforms [S1, unofficial]. It decomposes the signal into a family of functions, which are the translation and dilation of a unique function $\psi(x)$ [S1, unofficial]. The transform has roots in signal processing and pure mathematics [S2, unofficial]. It is used in image compression, solving partial differential equations, and integrating radiosity kernels [S2, unofficial]. The lifting scheme is used to construct wavelets for efficiently modeling massive volumetric data sets [S2, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] Multiresolution modeling for scientific visualization, https://escholarship.org/uc/item/5317385v, fetched 2026-10-26 (unofficial)
