# Local Oracle

Also called: Wavelet Space Filter

## Overview
The Local Oracle is a compression method that controls approximation error locally in specific regions of interest (ROI) using a spatial weighting function [S1, unofficial]. It operates by projecting a spatial weighting function into wavelet space to accentuate or suppress individual coefficients based on their spatial support [S1, unofficial]. Unlike global thresholding which applies a uniform cutoff, the Local Oracle allows for variable precision across the dataset, prioritizing fidelity in defined areas [S1, unofficial]. It builds on [Wavelet Compression (Haar Basis)](5779-wavelet-compression-haar-basis.md) by adding a spatially adaptive filtering stage.

## How it works
**Inputs and Outputs:**
*   Input: Wavelet coefficients $d[i,m]$ from a forward transform, and a spatial weighting function $W(x)$ defining the ROI [S1, unofficial].
*   Output: Filtered wavelet coefficients ready for storage or transmission [S1, unofficial].

**Data Structures:**
*   Coefficient array: Stores wavelet coefficients indexed by position $i$ and scale level $m$.
*   Score array: `score[i,m]` accumulates the conditional approximation error for rejecting coefficient $d[i,m]$ [S1, unofficial].

**Steps:**
1.  **Project Weighting Function:** Map the spatial weighting function $W(x)$ into wavelet space. This accounts for the Heisenberg uncertainty bound and the dyadic scale of the wavelets [S1, unofficial].
2.  **Compute Center in Wavelet Space:** For a Gaussian weighting function centered at $x_0$ in spatial domain, compute the center in wavelet space at level $m$ using the scaling property:
    $$ x_0^m = \frac{x_0}{2^m} $$
    $$ y_0^m = \frac{y_0}{2^m} $$
    [S1, unofficial]
3.  **Calculate Influence:** Determine the influence of each wavelet coefficient $d[i,m]$ on the ROI. This involves computing the overlap between the basis function's support region and the projected weighting function [S1, unofficial].
4.  **Accumulate Scores:** For each coefficient, update its score based on the weighted error introduced by rejecting it:
    ```python
    # Pseudocode for score accumulation
    # increment[i,m] is the weighted error contribution of coefficient i at level m
    score[i,m] = score[i,m] + increment[i,m]
    ```
    [S1, unofficial]
5.  **Filter Coefficients:** Compare `score[i,m]` against a threshold. If the score (representing the conditional approximation error) is below the threshold, the coefficient is retained; otherwise, it is set to zero (neglected) [S1, unofficial][S2, unofficial].

**Parameters:**
*   Threshold value: User-defined cutoff for the score or coefficient magnitude. Coefficients with scores below this value are neglected [S2, unofficial].
*   Gaussian width: Controls the spread of the ROI weighting function [S1, unofficial][S2, unofficial].

**Pitfalls:**
*   **Support Region Ignorance:** Early versions of the method ignored the support regions of individual basis functions. This caused artifacts when rejecting wavelets that extended into the region of interest [S1, unofficial].
*   **Solution:** The method must be extended by explicitly computing the support of the basis functions and using endpoint-interpolating B-splines to handle boundaries correctly [S1, unofficial].
*   **Boundary Discontinuities:** Standard zero-padding introduces artificial singularities at endpoints. Periodization or special boundary elements should be used to avoid severe errors [S1, unofficial].

## Subtopics

### Global and Local Thresholding Compression
Performs lossy compression on wavelet coefficients by discarding values below a threshold, leveraging the localization properties of B-spline wavelets [S2, unofficial]. It decomposes data, applies a thresholding filter (global or locally weighted by a Gaussian), and neglects coefficients below the defined value [S2, unofficial]. The local variant uses a user-defined Gaussian function to weight the thresholding, similar to the Local Oracle but potentially simpler in implementation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** The time complexity of the oracle is $O(N^2)$ and applies only to forward compression [S1, unofficial].
*   **Memory Complexity:** Requires storage for the original coefficients and the score array of the same size [S1, unofficial].
*   **Strengths:** Allows precise control over approximation error in specific regions, enhancing visual quality where it matters most [S1, unofficial].
*   **Limitations:** Higher computational cost due to score accumulation; sensitive to boundary handling and basis function support calculations [S1, unofficial].
*   **When to Use:** When regions of interest require higher fidelity than the rest of the volume, and computational resources allow for $O(N^2)$ preprocessing.

## Background
The Local Oracle was introduced as a method to control approximation locally in interesting regions by leveraging the spatial localization of wavelet basis functions [S1, unofficial]. The initial approach by Gross and others employed a Gaussian weighting but failed to consider basis function support regions, leading to artifacts [S1, unofficial]. The method was later extended to compute support regions and use endpoint-interpolating B-splines [S1, unofficial]. It corresponds to a filter operation in wavelet space, analogous to frequency domain filters, hence the alias "Wavelet Space Filter" [S1, unofficial]. B-spline wavelets are noted for approaching the Heisenberg uncertainty limit, with cubic B-splines having a variance product about 5% of the theoretical limit [S2, unofficial].

## Sources
[S1] Multiresolution representation and compression of surfaces and volumes, https://doi.org/10.3929/ethz-a-004086690, fetched 2026-10-26 (unofficial)
[S2] Ray–tracing of multiresolution B–Spline volumes, https://doi.org/10.3929/ethz-a-006651431, fetched 2026-10-26 (unofficial)
