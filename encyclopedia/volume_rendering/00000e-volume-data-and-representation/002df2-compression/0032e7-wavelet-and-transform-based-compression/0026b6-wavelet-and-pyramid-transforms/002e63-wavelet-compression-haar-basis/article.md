# Wavelet Compression (Haar Basis)

Also called: Global Oracle (Orthogonal), Haar Wavelet Linear Algebra Approach, Matrix-based Haar Transform, Multiresolution Image Compression, Wavelet Coefficient Thresholding, Wavelet Thresholding

## Overview
Wavelet Compression (Haar Basis) is a lossy or lossless compression method that represents volume data using a smaller set of coefficients derived from the Haar wavelet basis [S1, unofficial]. It solves the problem of reducing memory footprint and bandwidth for volume rendering by exploiting the sparsity of wavelet coefficients, where most energy is concentrated in a few large coefficients [S1, S3, unofficial]. Unlike sibling methods that may use biorthogonal filters or packet transforms, this method relies on the orthogonal Haar basis, allowing for a globally optimal compression strategy via coefficient thresholding (the "Global Oracle") that minimizes the $L^2$ approximation error for a given number of retained coefficients [S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A 3D volume data set $V$ (or 2D slices/vectors within it).
- Output: A compressed bitstream of wavelet coefficients, optionally with a threshold or error tolerance parameter.

**Data Structures:**
- **Coefficient Vector:** A 1D array or flattened structure storing the Haar wavelet coefficients. For multi-level decomposition, coefficients are typically organized by scale (approximation coefficients at the coarsest level, followed by detail coefficients at finer levels).
- **Haar Matrix ($W$ or $A_i$):** A sparse orthogonal matrix representing the transform. For a vector of length $N=2^k$, the matrix is constructed from the Kronecker product of 1D Haar matrices or applied recursively [S2, unofficial].

**Steps:**

1.  **Preprocessing (Optional):** Normalize density values to a standard range (e.g., 0–255) if histograms are outside expected bounds [S2, unofficial]. Apply a density filter to zero out values below $d_1$ or above $d_3$ if specific visualization goals require it [S2, unofficial].

2.  **Forward Transform:**
    Apply the Haar wavelet transform to the data. This can be done using a separable filter bank or matrix multiplication [S2, unofficial].
    - **Matrix Approach:** For a 1D input vector $\tilde{x}_0$, compute $\tilde{X}_1 = A_1 \tilde{x}_0$, where $A_1$ is the Haar matrix for one level [S2, unofficial]. For multi-level compression, the total transform matrix is $W = A_1 A_2 \dots A_n$, and the result is $\tilde{X}_1 = W \tilde{x}_0$ [S2, unofficial].
    - **Separable Approach:** For a 3D volume, apply the 1D Haar transform along each axis. For a 2D matrix $P$, apply row transformation $Q = P W$ and column transformation $T = W^T P W$ [S2, unofficial].
    - **Integer Packing Variant:** To avoid floating-point arithmetic, use unsigned 32-bit integers. The unused bits of the 32-bit word handle intermediate overflows/underflows, maintaining precision without loss [S1, unofficial].

3.  **Coefficient Sorting and Thresholding (Global Oracle):**
    - Sort all wavelet coefficients by their absolute magnitude (descending) [S3, unofficial].
    - Determine the set of coefficients to retain based on a user-specified error tolerance $\epsilon$ (for lossless, $\epsilon = 0$) or a target compression ratio [S1, S3, unofficial].
    - Reject the $K$ smallest coefficients. The optimal strategy for orthogonal bases is to keep the $N-K$ largest coefficients [S3, unofficial].

4.  **Encoding:**
    - Store the indices and values of the retained coefficients.
    - Apply entropy coding (e.g., Huffman, Arithmetic, Run-Length) to the quantized or raw coefficients to produce the compressed bitstream [Parent].

5.  **Reconstruction (Decompression):**
    - Decode the bitstream to retrieve the retained coefficients and their positions.
    - Reconstruct the full coefficient vector by inserting zeros at the positions of rejected coefficients.
    - Apply the inverse transform: $\tilde{x}_0 = W^{-1} \tilde{X}_1$, where $W^{-1} = A_n^{-1} \dots A_2^{-1} A_1^{-1}$ [S2, unofficial].
    - For the integer packing variant, perform the inverse integer operations to recover the original data [S1, unofficial].

**Formulas:**
- **Transform:** $\tilde{X}_1 = W \tilde{x}_0$ [S2, unofficial].
- **Inverse Transform:** $\tilde{x}_0 = W^{-1} \tilde{X}_1$ [S2, unofficial].
- **Approximation Error:** The $L^2$ error of the approximation is the sum of the squares of the rejected coefficients:
  $$ ||f - f'||^2 = \sum_{i=K+1}^N |c_i|^2 $$
  where $c_i$ are the sorted coefficients and $N$ is the total count [S3, unofficial].
- **Optimality Condition:** For orthogonal bases, keeping the largest coefficients minimizes this error [S3, unofficial].

**Parameters:**
- $\epsilon$: User-specified error tolerance. $\epsilon = 0$ for lossless compression [S1, unofficial].
- $K$: Number of coefficients to reject.
- $d_1, d_2, d_3, d_4$: Density threshold values for preprocessing filtering [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Boundary Conditions:** Standard Haar transforms on bounded intervals can introduce artifacts due to discontinuities at endpoints if zero-padding is used. Periodization or special boundary elements (e.g., endpoint-interpolating B-splines) should be considered for bounded domains [S3, unofficial].
- **Support Regions:** When using local oracles (see Subtopics), ignoring the support regions of wavelet basis functions can cause artifacts when rejecting coefficients that extend into regions of interest [S3, unofficial].
- **Precision:** Floating-point operations may suffer from precision loss. Using integer arithmetic with sufficient bit-width (e.g., 32-bit) can mitigate this [S1, unofficial].

## Subtopics

### [3D Non-standard Haar Wavelet Decomposition with Integer Packing](6693-3d-non-standard-haar-wavelet-decomposition-with-integer-pack.md)
This method performs a 3D Haar wavelet decomposition using unsigned 32-bit integers to store coefficients, replacing floating-point arithmetic to leverage faster integer operations without loss of precision. It differs from standard float-based wavelet compression by utilizing the unused bits of a 32-bit integer word to handle intermediate overflows and underflows, allowing for a fixed-point hierarchy that occupies the same memory footprint as the original volume.

### Local Oracle
A compression method that controls approximation error locally in specific regions of interest using a spatial weighting function [S3, unofficial]. It projects the spatial weighting function into wavelet space, accounting for the Heisenberg bound and dyadic scale. The center of the Gaussian in wavelet space at level $m$ is computed as $x_0^m = x_0 / 2^m$ and $y_0^m = y_0 / 2^m$ [S3, unofficial]. A key pitfall is that early versions ignored basis function support regions, causing artifacts when rejecting wavelets that extend into the region of interest [S3, unofficial].

## Operating characteristics
- **Time Complexity:** The forward and inverse transforms are $O(N)$ for $N$ data points using the fast wavelet transform. Sorting coefficients for the global oracle takes $O(N \log N)$ [S3, unofficial].
- **Memory Complexity:** The compressed size depends on the number of retained coefficients. In the integer packing variant, the memory footprint is identical to the original volume if all coefficients are stored, but can be reduced significantly if only non-zero or large coefficients are stored [S1, unofficial].
- **Performance:** The method is effective for volumes with smooth regions and sharp edges, as the Haar basis is good at capturing discontinuities [S2, unofficial]. However, it may produce blocky artifacts in smooth gradients compared to smoother wavelets (e.g., B-spline) [S3, unofficial].
- **Strengths:** Simple implementation, orthogonal basis allows for optimal $L^2$ error minimization via simple thresholding, integer arithmetic support for speed [S1, S3, unofficial].
- **Limitations:** Poor frequency localization compared to smoother wavelets, potential boundary artifacts [S3, unofficial].

## Background
The method is based on the work of Stollnitz et al. in computer graphics, applying multiresolution decomposition using wavelets to volume data [S1, unofficial]. The Haar wavelet is one of the simplest wavelet bases, defined by piecewise constant functions. The "Global Oracle" concept for orthogonal wavelets was formalized in the context of optimal compression by rejecting the smallest coefficients to minimize error [S3, unofficial]. The matrix-based linear algebra approach provides an alternative view of the transform, useful for understanding the invertibility and structure of the decomposition [S2, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] Volumetric Medical Images Visualization on Mobile Devices, http://hdl.handle.net/2099.1/11308, fetched 2026-10-26 (unofficial)
[S3] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
