# Third Order Elimination (Scaling)

Also called: Scaling, PSR Scaling

## Overview
Third Order Elimination (Scaling) is a quadratic interpolation method that derives a second-order polynomial from a cubic Hermite basis by scaling the input derivatives to force the third-order coefficient ($a_3$) to zero [S1]. It solves the problem of approximating a smooth field with $C^1$ continuity while reducing computational complexity from cubic to quadratic, specifically for cases where the underlying function is locally linear or where derivative cancellation is rare [S1]. It differs from [Piecewise Quadratic Hermite Interpolation](4001-piecewise-quadratic-hermite-interpolation.md) and [Least-Squares Derivatives](11066-b-spline-field-interpolation.md) by using a multiplicative scaling factor rather than an additive offset or least-squares minimization [S1].

## How it works
**Inputs:**
*   $v_0, v_1$: Scalar values at grid nodes $t=0$ and $t=1$.
*   $d_0, d_1$: Derivatives (gradients projected onto the axis) at $t=0$ and $t=1$.

**Outputs:**
*   Quadratic polynomial coefficients $a_0, a_1, a_2$ for $p(t) = a_0 + a_1 t + a_2 t^2$.

**Data Structures:**
*   No specific layout required beyond standard scalar storage for values and derivatives.

**Steps:**
1.  Compute the scaling factor $s$ using the values and derivatives:
    $$ s = \frac{2v_1 - 2v_0}{d_0 + d_1} $$
2.  Scale the derivatives to obtain effective derivatives $d'_0$ and $d'_1$:
    $$ d'_0 = s \cdot d_0 $$
    $$ d'_1 = s \cdot d_1 $$
3.  Determine the quadratic coefficients based on the value constraints and scaled derivatives. The method ensures $p(0)=v_0$, $p(1)=v_1$, and the cubic term vanishes. The coefficients are derived from the condition that the third-order term in the cubic Hermite expansion is eliminated by this scaling [S1].
    *   $a_0 = v_0$
    *   $a_1 = d'_0$
    *   $a_2 = \frac{v_1 - v_0 - d'_0}{1}$ (Derived from $p(1) = a_0 + a_1 + a_2 = v_1$)
    *   *Note:* The source explicitly defines the scaling step and the instability condition but implies the resulting quadratic form by stating the cubic term vanishes. The standard quadratic Hermite form with scaled derivatives is used.

**Pitfalls:**
*   **Instability:** The solution becomes unstable if $d_0 + d_1 \approx 0$ (derivatives cancel each other out) [S1]. This occurs when the gradient changes sign or is near zero in a way that sums to zero.
*   **Context Dependency:** The instability is less problematic for functions like the indicator function in PSR, which have steep gradients and unlikely opposite signs in partial derivatives [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def third_order_elimination_scaling(v0, v1, d0, d1):
    """
    Computes quadratic interpolation coefficients using Third Order Elimination (Scaling).
    
    Args:
        v0 (float): Value at t=0.
        v1 (float): Value at t=1.
        d0 (float): Derivative at t=0.
        d1 (float): Derivative at t=1.
        
    Returns:
        tuple: (a0, a1, a2) coefficients for p(t) = a0 + a1*t + a2*t^2
    """
    # Step 1: Compute scaling factor
    denominator = d0 + d1
    
    # Guard against division by zero or near-zero
    if abs(denominator) < 1e-9:
        # Fallback to linear interpolation or handle error
        # [G] Standard practice is to clamp or use linear if derivative sum is zero
        # However, source notes this is unstable. We return None or handle externally.
        return None 

    s = (2.0 * v1 - 2.0 * v0) / denominator
    
    # Step 2: Scale derivatives
    d0_scaled = s * d0
    # d1_scaled = s * d1 # Not explicitly needed for quadratic coeff calculation if using boundary conditions
    
    # Step 3: Compute quadratic coefficients
    # p(0) = a0 = v0
    a0 = v0
    
    # p'(0) = a1 = d0_scaled
    a1 = d0_scaled
    
    # p(1) = a0 + a1 + a2 = v1  => a2 = v1 - a0 - a1
    a2 = v1 - a0 - a1
    
    return (a0, a1, a2)
```

## Subtopics

### Third Order Elimination (Offset)
A quadratic method obtained by adding an offset $o$ to derivatives to eliminate the third order coefficient $a_3$ [S1]. It solves $2v_0 - 2v_1 + (d_0 + o) + (d_1 + o) = 0$, yielding $o = (2v_1 - 2v_0 - d_0 - d_1) / 2$ [S1]. This solution is equivalent to the Least-Squares Derivatives method (equation 11 in [S1]) [S1]. It avoids the division instability of Scaling when $d_0 + d_1 \approx 0$ but may introduce different bias characteristics [S1].

## Operating characteristics
*   **Complexity:** $O(1)$ arithmetic operations. Slightly more expensive than linear interpolation due to the division and scaling, but cheaper than full cubic Hermite evaluation.
*   **Memory:** Requires storage of values and derivatives at grid nodes.
*   **Strengths:** Reduces polynomial degree from 3 to 2, simplifying evaluation. Effective for functions with steep gradients where derivative cancellation is rare (e.g., indicator functions in PSR) [S1].
*   **Limitations:** Numerically unstable when $d_0 + d_1 \approx 0$ [S1]. Not suitable for general-purpose interpolation where derivative signs may vary arbitrarily without fallback mechanisms.
*   **Performance:** Implemented in the PSR code for interpolating the indicator function [S1].

## Background
Originated in the context of accurate isosurface interpolation with Hermite data [S1]. It is part of a family of methods comparing linear, quadratic, and cubic interpolations for implicit surfaces [S1]. The method is specifically noted as being implemented in the PSR (Presumably "Piecewise Smooth Reconstruction" or similar context from the source) code [S1].

## Sources
[S1] Accurate Isosurface Interpolation with Hermite Data, https://ieeexplore.ieee.org/document/7335492/, fetched 2026-11-05
