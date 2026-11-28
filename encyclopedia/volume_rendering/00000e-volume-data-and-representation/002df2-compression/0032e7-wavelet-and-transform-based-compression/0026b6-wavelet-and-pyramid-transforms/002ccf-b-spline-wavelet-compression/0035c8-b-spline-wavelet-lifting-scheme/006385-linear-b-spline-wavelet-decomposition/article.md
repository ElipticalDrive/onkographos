# Linear B-spline Wavelet Decomposition

Also called: Inverse Lifting Operations for B-spline Wavelets

## Overview
Linear B-spline wavelet decomposition is the analysis step of the linear B-spline wavelet lifting scheme, transforming a signal into approximation and detail coefficients [S1]. It separates low-frequency content from high-frequency details using inverse lifting operations, avoiding the over- and undershoots (and resulting topological changes in isosurfaces) caused by standard subdivision wavelets [B-spline Wavelet Lifting Scheme](7585-b-spline-wavelet-lifting-scheme.md). It is distinguished from its sibling [2D B-spline Wavelet Masks (Quadtree)](7753-2d-b-spline-wavelet-masks-quadtree.md) by its specific 1D lifting formulation for linear splines, whereas the sibling addresses 2D mesh structures with arbitrary polygons.

## How it works
**Inputs and Outputs:**
*   Input: A signal $s$ and a wavelet coefficient array $w$.
*   Output: Updated $s$ (approximation coefficients) and $w$ (detail coefficients).

**Data Structures:**
Not given in the sources.

**Steps:**
The decomposition is performed by applying the inverse of the lifting operations in reverse order [S1]. For the linear B-spline case, the process consists of two sequential lifting steps applied to the current signal $s$ to produce the next level's approximation $s'$ and detail $w'$ [S1].

1.  **First Lifting Step (Prediction):**
    The detail coefficients $w$ are computed using the inverse prediction lift:
    $$ w = \text{lift}(-\sim, l) $$
    Here, $\text{lift}(\sim, l)$ represents the predicted value based on the lifting scheme parameters, and $\sim$ denotes the operation of predicting the missing sample from its neighbors [S1].

2.  **Second Lifting Step (Update):**
    The approximation coefficients $s$ are updated using the inverse update lift:
    $$ s = \text{lift}(\sim, l) $$
    This step preserves moments (such as the mean) and completes the decomposition [S1].

**Parameters:**
*   $l$: The lifting step parameter. The source does not give a specific numerical value for $l$ in the linear case, but notes that cubic cases use $a, b$ coefficients [S1].

**Edge Cases and Pitfalls:**
The source notes that the lifting operations are well-defined for extraordinary vertices and arbitrary polygons in 2D extensions, but does not specify 1D boundary handling, precision constraints, or degenerate input behavior [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def linear_b_spline_wavelet_decomposition(signal, l):
    """
    Decomposes a 1D signal into approximation and detail coefficients
    using the inverse lifting operations for linear B-splines [S1].
    """
    n = len(signal)
    approx = [0.0] * (n // 2)
    detail = [0.0] * (n // 2)
    
    # The source defines decomposition as applying inverse lifting operations in reverse order [S1].
    # Explicit 1D formulas for the linear case are not provided in the sources [S1].
    
    # Step 1: Compute detail coefficients w using the inverse prediction lift
    # [G] Implementer note: The source does not specify the exact 1D prediction formula.
    # Standard linear B-spline lifting predicts odd samples from even neighbors.
    for i in range(n // 2):
        # detail[i] = signal[2*i + 1] - lift(-~, l)
        pass
        
    # Step 2: Compute approximation coefficients s using the inverse update lift
    # [G] Implementer note: The source does not specify the exact 1D update formula.
    # Standard linear B-spline lifting updates even samples using the detail coefficients.
    for i in range(n // 2):
        # approx[i] = signal[2*i] + lift(~, l)
        pass
        
    return approx, detail
```

## Subtopics

### [Lifting scheme for 1/2 subdivision](19213-lifting-scheme-for-1-2-subdivision.md)
Applies this decomposition specifically to 1/2 subdivision, handling hypervolume, volume, face, and edge cases with automatic boundary management.

## Operating characteristics
Not given in the sources.

## Background
Originated in the context of wavelet representation of contour sets and volume data compression [S1]. The method is part of a family of B-spline wavelet constructions that use lifting schemes to ensure best approximation properties [S1].

## Sources
[S1] Wavelet Representation of Contour Sets, https://digital.library.unt.edu/ark:/67531/metadc1405239/?q=volume data compression wavelet, fetched 2026-10-28
