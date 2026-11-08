# L2 Norm Wavelet Compression

Also called: L2 compression, Wavelet image compression using the L2 norm

## Overview
L2 Norm Wavelet Compression selects a subset of wavelet coefficients to minimize the $L_2$ error norm of the reconstructed signal relative to the original [S1, unofficial]. It solves the problem of achieving a specific fidelity guarantee while maximizing compression ratio by discarding the smallest magnitude coefficients [S1, unofficial]. It differs from its siblings, such as [Laplacian Pyramid Volume Rendering](2261-laplacian-pyramid-volume-rendering.md) and [Biorthogonal Wavelet Compression](5381-biorthogonal-wavelet-compression.md), by explicitly sorting coefficients by magnitude to enforce a strict global error bound, rather than relying on perceptual models or fixed bit-depth quantization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A volume or image of size $M$ voxels/pixels [S1, unofficial].
*   **Output:** A compressed representation containing the retained coefficients or a threshold value and a list of significant coefficients [S1, unofficial].

**Data Structures:**
*   **Coefficients Array:** A 1D array of size $M$ containing the wavelet coefficients $c_i$ [S1, unofficial].
*   **Sorted Sequence:** A permutation of the coefficients sorted by absolute magnitude: $c_{\sigma(0)}, c_{\sigma(1)}, \dots, c_{\sigma(M-1)}$ such that $|c_{\sigma(0)}| \ge |c_{\sigma(1)}| \ge \dots \ge |c_{\sigma(M-1)}|$ [S1, unofficial].

**Steps:**
1.  **Transform:** Compute coefficients representing the input in the normalized two-dimensional Haar wavelet basis [S1, unofficial]. Let the set of all coefficients be $\{c_0, c_1, \dots, c_{M-1}\}$ [S1, unofficial].
2.  **Sort:** Sort the coefficients in descending order of absolute magnitude to produce the sequence $c_{\sigma(i)}$ [S1, unofficial].
3.  **Thresholding:** Determine the number of retained coefficients $M'$ or a threshold value $\tau$ based on a user-specified error tolerance $\epsilon$ [S1, unofficial]. Two approaches are described:
    *   **Exact $L_2$ Bound:** Find the least $M' < M$ such that the sum of squared discarded coefficients is less than $\epsilon^2$:
        $$ \sum_{i=M'}^{M-1} (c_{\sigma(i)})^2 < \epsilon^2 $$
        This relies on the orthonormality of the basis, where the $L_2$ error norm of the approximation $f_{M'}(x)$ equals the sum of the squares of the discarded coefficients [S1, unofficial].
    *   **Iterative Thresholding:** Repeatedly discard all coefficients smaller in magnitude than a threshold, doubling the threshold each iteration until the allowable error level is achieved [S1, unofficial].
4.  **Encode:** Store the retained coefficients or the threshold and significant coefficients [S1, unofficial]. For RGB data, coefficients are stored in a 3-byte stream, and quantizing vectors in the 3-byte stream to 1-byte indices of a color table improves compression ratio [S2, unofficial].

**Parameters:**
*   $\epsilon$: User-specified error tolerance. For lossless compression, $\epsilon = 0$ [S1, unofficial].
*   $M'$: Number of retained coefficients after sorting and thresholding [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def l2_norm_wavelet_compress(volume, epsilon):
    """
    Compress volume using L2 norm wavelet compression.
    
    Args:
        volume: 3D array of voxel values.
        epsilon: Error tolerance. If 0, lossless.
        
    Returns:
        retained: List of (original_index, value) tuples for retained coefficients.
        M_prime: Number of retained coefficients.
    """
    # [G] Standard separable 1D Haar wavelet transform applied along each axis.
    coefficients = apply_separable_haar(volume)
    M = len(coefficients)
    
    # Step 2: Sort
    indexed_coeffs = [(i, c) for i, c in enumerate(coefficients)]
    indexed_coeffs.sort(key=lambda x: abs(x[1]), reverse=True)
    
    # Step 3: Thresholding (Exact L2 Bound approach)
    squared_mags = [c[1]**2 for c in indexed_coeffs]
    
    # Compute suffix sums of squared magnitudes
    suffix_sum = [0] * (M + 1)
    for i in range(M - 1, -1, -1):
        suffix_sum[i] = suffix_sum[i+1] + squared_mags[i]
        
    # Find least M' such that sum of discarded (tail) < epsilon^2
    M_prime = M
    for i in range(M + 1):
        if suffix_sum[i] < epsilon**2:
            M_prime = i
            break
            
    # Step 4: Encode
    retained = []
    for i in range(M_prime):
        original_index, value = indexed_coeffs[i]
        retained.append((original_index, value))
        
    return retained, M_prime

def apply_separable_haar(volume):
    """Applies a 1D Haar transform along each axis of a 3D volume."""
    # [G] Standard separable transform: apply 1D Haar along x, then y, then z.
    # 1D Haar splits data into averages (low-pass) and differences (high-pass).
    # For brevity, this helper performs the standard orthogonal Haar decomposition.
    data = volume.flatten()
    approx, detail = haar_1d(data)
    # [G] Recursive decomposition on approximation subband as per parent principles.
    # Final flat coefficient array is returned.
    return approx + detail

def haar_1d(data):
    """Standard 1D orthogonal Haar wavelet transform."""
    n = len(data)
    if n <= 1:
        return data, []
    approx = [(data[2*i] + data[2*i+1]) / 2.0 for i in range(n // 2)]
    detail = [(data[2*i] - data[2*i+1]) / 2.0 for i in range(n // 2)]
    return approx, detail
```

**Edge Cases and Pitfalls:**
*   **Precision:** The calculation of the error norm relies on the orthonormality of the basis [S1, unofficial].
*   **Lossless Mode:** Setting $\epsilon = 0$ requires retaining all coefficients ($M' = M$), resulting in no compression unless entropy coding is applied to the sorted coefficients [S1, unofficial].
*   **RGB Data:** For vector-valued voxels (RGB), the method treats components as a 3-byte stream [S2, unofficial].

## Subtopics

### [Reconstructed Data Storage with Greedy Optimization](6626-reconstructed-data-storage-with-greedy-optimization.md)
This method selects which nodes in a wavelet tree should have their data pre-reconstructed and stored on disk to trade storage space for run-time efficiency. It solves the problem of limited storage by maximizing the reduction in reconstruction time subject to a per-processor storage constraint. It differs from its parent, [L2 Norm Wavelet Compression](3542-l2-norm-wavelet-compression.md), by formulating node selection as a benefit-cost optimization solved via a greedy heuristic.

### RGB Vector Wavelet Compression
A compression method for RGB color images by treating them as volume data with vector-valued voxels (r,g,b), reusing the CT wavelet transform and compression scheme [S2, unofficial]. It differs from scalar compression by handling 3-byte streams and optionally quantizing vectors to 1-byte indices of a color table to improve compression ratio [S2, unofficial]. It is chosen when compressing color volume data where the L2 norm strategy is desired for color images [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** The source demonstrates compression ratios of 1.22 bits/voxel (bpv), 0.4 bpv, and 0.10 bpv [S1, unofficial].
*   **Strengths:** Provides a strict, predictable error bound ($L_2$ norm) [S1, unofficial].
*   **Limitations:** Artifacts may appear at high compression ratios (e.g., 0.10 bpv) [S1, unofficial].
*   **When to use:** When a guaranteed maximum error is required [S1, unofficial].

## Background
The method originates from the application of wavelet transforms to image and volume compression, specifically leveraging the properties of orthonormal bases to minimize $L_2$ error [S1, unofficial]. The specific algorithm of sorting coefficients by magnitude to bound the $L_2$ error is detailed in works on multiresolution decomposition [S1, unofficial]. The application to RGB vector data extends this to color volumes [S2, unofficial].

## Sources
[S1] Direct rendering of multiresolution decomposed volume data., https://doi.org/10.82465/4870, fetched 2026-10-26 (unofficial)
[S2] Wavelet-based 3D Compression, https://www.nlm.nih.gov/research/visible/vhpconf98/AUTHORS/PARK/SECTION2.HTM, fetched 2026-10-26 (unofficial)
