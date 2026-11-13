# Third Order Elimination (Scaling)

Also called: Scaling, SCALING

## Overview
Third Order Elimination (Scaling) is a quadratic interpolation method that derives a second-order polynomial from a cubic Hermite basis by scaling the input derivatives to force the cubic coefficient ($a_3$) to zero [S1]. It solves the problem of approximating a smooth implicit function with a lower-degree polynomial that preserves the endpoint values and adapts to local derivative magnitudes, distinguishing itself from fixed-basis methods like B-splines by dynamically adjusting the derivative weights [S1]. Unlike the Offset variant, it uses multiplicative scaling, which makes it sensitive to the sum of the derivatives [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Endpoint values $v_0 = p(0)$ and $v_1 = p(1)$; endpoint derivatives $d_0 = p'(0)$ and $d_1 = p'(1)$; local coordinate $t \in [0, 1]$.
*   **Output:** Interpolated value $p(t)$.

**Data Structures:**
No special data structures are required beyond the scalar values $v_0, v_1, d_0, d_1$.

**Method Steps:**
1.  Compute the scaling factor $s$ using the endpoint values and derivatives.
2.  Scale the derivatives: $d'_0 = s \cdot d_0$ and $d'_1 = s \cdot d_1$.
3.  Evaluate the cubic Hermite interpolation polynomial using the original values $v_0, v_1$ and the scaled derivatives $d'_0, d'_1$.

**Formulas:**
The cubic Hermite polynomial is defined as:
$$ p(t) = h_{00}(t)v_0 + h_{10}(t)v_1 + h_{01}(t)d'_0 + h_{11}(t)d'_1 $$
where the basis functions are:
$$ h_{00}(t) = 2t^3 - 3t^2 + 1 $$
$$ h_{10}(t) = -2t^3 + 3t^2 $$
$$ h_{01}(t) = t^3 - 2t^2 + t $$
$$ h_{11}(t) = t^3 - t^2 $$

The scaling factor $s$ is calculated as [S1]:
$$ s = \frac{2v_1 - 2v_0}{d_0 + d_1} $$

Substituting $s$ into the derivative terms effectively eliminates the $a_3$ coefficient of the underlying cubic polynomial, resulting in a quadratic approximation [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def third_order_elimination_scaling(v0, v1, d0, d1, t):
    # Calculate scaling factor s
    denominator = d0 + d1
    
    # Guard against division by zero or near-zero
    if abs(denominator) < 1e-9:
        # Fallback to linear interpolation if derivatives cancel out
        # [G] Standard practice to avoid NaN/Inf; source notes instability here
        return v0 + (v1 - v0) * t
    
    s = (2.0 * v1 - 2.0 * v0) / denominator
    
    # Scale derivatives
    d0_scaled = s * d0
    d1_scaled = s * d1
    
    # Cubic Hermite basis functions
    t2 = t * t
    t3 = t2 * t
    
    h00 = 2.0 * t3 - 3.0 * t2 + 1.0
    h10 = -2.0 * t3 + 3.0 * t2
    h01 = t3 - 2.0 * t2 + t
    h11 = t3 - t2
    
    # Interpolated value
    p = h00 * v0 + h10 * v1 + h01 * d0_scaled + h11 * d1_scaled
    
    return p
```

**Parameters:**
*   $t$: Local coordinate in $[0, 1]$.
*   No other tunable parameters.

**Edge Cases and Pitfalls:**
*   **Instability:** The solution becomes unstable if $d_0 + d_1 \approx 0$ (derivatives cancel each other out) [S1]. This occurs when the implicit function has partial derivatives with opposite signs at the endpoints.
*   **Mitigation:** In contexts like the PSR code for indicator functions, this instability is less problematic because the indicator function has a steep gradient near the isosurface, making opposite-sign derivatives unlikely [S1]. However, a robust implementation should handle the zero-denominator case, typically by falling back to linear interpolation or the Offset variant [G].

## Subtopics

### Third Order Elimination (Offset)
A quadratic method obtained by adding an offset $o$ to the derivatives to eliminate the third-order coefficient $a_3$ [S1]. It solves for $o$ such that $2v_0 - 2v_1 + (d_0 + o) + (d_1 + o) = 0$, yielding $o = \frac{1}{2}(2v_1 - 2v_0 - d_0 - d_1)$ [S1]. This solution is equivalent to the Least-Squares Derivatives (LSDERIV) method [S1]. It is preferred when derivatives might cancel out, as it avoids the division instability of the Scaling variant [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample. Requires a few multiplications and additions.
*   **Memory Complexity:** $O(1)$ auxiliary storage.
*   **Strengths:** Produces a quadratic approximation that is computationally cheaper than full cubic Hermite while maintaining $C^0$ continuity and adapting to derivative magnitudes [S1]. It is specifically designed for interpolating indicator functions in implicit surface reconstruction [S1].
*   **Limitations:** Numerically unstable when $d_0 + d_1 \approx 0$ [S1]. Not suitable for general-purpose field interpolation where derivative signs are unpredictable without fallback logic.
*   **When to use:** When interpolating indicator functions or similar fields where derivatives are unlikely to have opposite signs, and a quadratic approximation is sufficient [S1].
*   **When not to use:** When derivative stability is critical or derivatives may cancel out; use the Offset variant or Least-Squares Derivatives instead [S1].

## Background
The method originates from research on accurate isosurface interpolation using Hermite data, specifically within the context of the PSR (Piecewise Smooth Reconstruction) code [S1]. It was developed as an alternative to cubic interpolation to reduce computational cost while maintaining accuracy for implicit surface extraction [S1]. The approach is part of a family of "Third Order Elimination" techniques that reduce a cubic polynomial to quadratic by manipulating derivative constraints [S1].

## Sources
[S1] Accurate Isosurface Interpolation with Hermite Data, https://ieeexplore.ieee.org/document/7335492/, fetched 2026-11-05
