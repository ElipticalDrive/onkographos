# Linear B-spline Wavelet Decomposition

Also called: Inverse Lifting Operations for B-spline Wavelets

## Overview
Linear B-spline wavelet decomposition is the analysis step of the linear B-spline wavelet lifting scheme, transforming a signal into approximation and detail coefficients. It solves the problem of multiresolution analysis by separating low-frequency content from high-frequency details without the topological artifacts (over/undershoots) inherent in standard subdivision wavelets [S1]. It is distinguished from its sibling [2D B-spline Wavelet Masks (Quadtree)](7753-2d-b-spline-wavelet-masks-quadtree.md) by its specific 1D lifting formulation for linear splines, whereas the sibling addresses 2D mesh structures with arbitrary polygons.

## How it works
**Inputs and Outputs:**
*   Input: A signal $s$ and a wavelet coefficient array $w$ (initially empty or zeroed).
*   Output: Updated $s$ (approximation coefficients) and $w$ (detail coefficients).

**Data Structures:**
The method operates on scalar values or vectors representing vertex positions in a mesh. No complex bit-packing is specified; standard floating-point or integer types are used [S1].

**Steps:**
The decomposition is performed by applying the inverse of the lifting operations in reverse order. For the linear B-spline case, the process consists of two sequential lifting steps applied to the current signal $s$ to produce the next level's approximation $s'$ and detail $w'$ [S1].

1.  **First Lifting Step (Prediction):**
    The detail coefficients $w$ are computed by predicting the even-indexed samples from the odd-indexed samples (or vice versa, depending on the specific subdivision convention, but the formula uses the lifting operator). The formula is:
    $$ w = s - \text{lift}(\sim, l) $$
    Here, $\text{lift}(\sim, l)$ represents the predicted value based on the lifting scheme parameters. In the context of linear B-splines, this typically involves averaging neighbors. The symbol $\sim$ denotes the operation of predicting the missing sample from its neighbors.

2.  **Second Lifting Step (Update):**
    The approximation coefficients $s$ are updated using the newly computed detail coefficients to preserve moments (such as the mean). The formula is:
    $$ s = s + \text{lift}(\sim, l) $$
    Note: The source notation $s - \text{lift}(\sim, l)$ for $w$ and $s - \text{lift}(\sim, l)$ for $s$ in the raw text appears to be a typo or specific notation where the second operation uses the *updated* $s$ or a different sign. Standard lifting schemes usually follow:
    $$ w_i = s_{2i+1} - P(s_{2i}) $$
    $$ s_{2i} = s_{2i} + U(w_i) $$
    However, adhering strictly to the source [S1] which states:
    $$ w = \text{lift}(-\sim, l) $$
    $$ s = \text{lift}(\sim, l) $$
    We interpret this as:
    1. Compute $w$ using the inverse of the prediction lift.
    2. Compute $s$ using the inverse of the update lift.

    *Clarification from Source [S1]:* The text explicitly lists:
    $$ w = \text{lift}(-\sim, l) $$
    $$ s = \text{lift}(\sim, l) $$
    This implies the decomposition is the exact inverse of the reconstruction. If reconstruction is $s_{new} = s_{old} + \text{lift}(\sim, l)$ and $w_{new} = s_{old} - \text{lift}(\sim, l)$, then decomposition reverses these.

**Parameters:**
*   $l$: The lifting step parameter. For linear B-splines, this is typically fixed to ensure the "best approximation" property. The source does not give a specific numerical value for $l$ in the linear case, but cubic cases use $a, b$ coefficients. For linear, the prediction is usually a simple average.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_b_spline_wavelet_decomposition(signal):
    """
    Decomposes a 1D signal into approximation and detail coefficients
    using the inverse lifting operations for linear B-splines.
    
    Args:
        signal: List of float values representing the input signal.
        
    Returns:
        approx: List of approximation coefficients.
        detail: List of detail coefficients.
    """
    n = len(signal)
    # Ensure even length for standard dyadic decomposition
    if n % 2 != 0:
        # [G] Pad with last element or mirror if not even, 
        # though source assumes well-defined indices.
        signal = signal + [signal[-1]]
        n = len(signal)

    approx = [0.0] * (n // 2)
    detail = [0.0] * (n // 2)

    # The source defines decomposition as the inverse of reconstruction.
    # Reconstruction for linear B-spline lifting typically:
    # 1. Predict odd from even: w[i] = s[2i+1] - (s[2i] + s[2i+2])/2
    # 2. Update even: s[2i] = s[2i] + w[i]/2 (to preserve mean)
    # Decomposition reverses this.
    
    # Step 1: Extract Detail (Inverse of Prediction)
    # w = s_odd - Predict(s_even)
    # In lifting notation: w = lift(-~, l)
    for i in range(n // 2):
        even_idx = 2 * i
        odd_idx = 2 * i + 1
        
        # Prediction: Average of neighbors for linear spline
        # If boundary, handle edge cases. Source mentions extraordinary vertices 
        # in 2D, but for 1D linear, standard averaging is implied.
        # [G] Assuming periodic or zero-boundary if not specified. 
        # Standard linear B-spline prediction uses average of adjacent even samples.
        # However, in a single step decomposition of a full signal:
        # We separate even and odd indices.
        
        # Let's assume the signal is interleaved or we are splitting it.
        # Standard lifting decomposition:
        # detail[i] = signal[2*i + 1] - (signal[2*i] + signal[2*i+2]) / 2
        # But we must handle boundaries.
        
        left = signal[2*i]
        right = signal[2*i + 2] if 2*i + 2 < n else left # [G] Default to left neighbor at boundary
        
        predicted = (left + right) / 2.0
        detail[i] = signal[2*i + 1] - predicted

    # Step 2: Update Approximation (Inverse of Update)
    # s_even = s_even + detail[i] / 2
    # In lifting notation: s = lift(~, l)
    for i in range(n // 2):
        even_idx = 2 * i
        # Update step adds half the detail to the even sample
        approx[i] = signal[2*i] + detail[i] / 2.0

    return approx, detail
```

## Subtopics

### [Lifting scheme for 1/2 subdivision](19213-lifting-scheme-for-1-2-subdivision.md)
Applies this decomposition specifically to 1/2 subdivision, handling hypervolume, volume, face, and edge cases with automatic boundary management.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for a signal of size $N$, as each sample is processed a constant number of times.
*   **Memory Complexity:** $O(N)$ to store the coefficients.
*   **Strengths:** Avoids topological changes in isosurfaces caused by over/undershoots in standard wavelets [S1].
*   **Limitations:** Requires careful handling of boundary conditions and extraordinary vertices in 2D extensions [S1].

## Background
Originated in the context of wavelet representation of contour sets and volume data compression. The method is part of a family of B-spline wavelet constructions that use lifting schemes to ensure best approximation properties [S1].

## Sources
[S1] Wavelet Representation of Contour Sets, https://digital.library.unt.edu/ark:/67531/metadc1405239/?q=volume data compression wavelet, fetched 2026-10-28
