# Third Order Elimination (Scaling)
Also called: SCALING
## Overview
Third Order Elimination (Scaling) is a quadratic interpolation method derived from cubic Hermite interpolation by scaling the input derivatives to force the cubic coefficient ($a_3$) to zero [S1]. It solves the problem of approximating a smooth field with a polynomial that retains quadratic continuity while strictly interpolating the function values at grid vertices [S1]. It differs from its sibling [Hermite Interpolation of Scalar Fields](9450-hermite-interpolation-of-scalar-fields.md) by discarding the cubic term entirely through derivative scaling, and from [Tricubic Interpolation](14963-tricubic-interpolation.md) by reducing the polynomial degree to ensure $C^1$ continuity without the complexity of cubic basis evaluation [S1].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Function values $v_0, v_1$ at two endpoints (e.g., grid vertices), and their corresponding derivatives $d_0, d_1$.
*   **Outputs:** A quadratic polynomial $p(t) = a_0 + a_1 t + a_2 t^2$ defined on the interval $t \in [0, 1]$, and the scaled derivatives used for evaluation.

**Data Structures:**
The method operates on scalar values. No complex data structures are required beyond the storage of $v_0, v_1, d_0, d_1$.

**Steps:**
1.  **Calculate Scaling Factor:** Compute the scaling factor $s$ using the function values and derivatives:
    $$ s = \frac{2v_1 - 2v_0}{d_0 + d_1} $$
2.  **Scale Derivatives:** Multiply the original derivatives by $s$ to obtain the effective derivatives for the quadratic polynomial:
    $$ d'_0 = s \cdot d_0 $$
    $$ d'_1 = s \cdot d_1 $$
3.  **Compute Polynomial Coefficients:** Determine the coefficients $a_0, a_1, a_2$ for the quadratic polynomial $p(t)$:
    *   $a_0 = v_0$
    *   $a_1 = d'_0$
    *   $a_2 = \frac{d'_1 - d'_0}{2}$
    *(Note: The source derives $a_2$ from the constraint that the cubic term vanishes. Specifically, for a cubic $p(t) = a_0 + a_1 t + a_2 t^2 + a_3 t^3$, setting $a_3=0$ and satisfying $p(0)=v_0, p(1)=v_1, p'(0)=d'_0, p'(1)=d'_1$ leads to the above. The source explicitly states $2v_0 - 2v_1 + s \cdot d_0 + s \cdot d_1 = 0$ is the condition for eliminating $a_3$ in the cubic context, which yields the formula for $s$ [S1].)*

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def third_order_elimination_scaling(v0, v1, d0, d1):
    """
    Computes quadratic interpolation coefficients by scaling derivatives
    to eliminate the cubic term.
    
    Args:
        v0: Value at t=0
        v1: Value at t=1
        d0: Derivative at t=0
        d1: Derivative at t=1
        
    Returns:
        a0, a1, a2: Coefficients for p(t) = a0 + a1*t + a2*t^2
    """
    denominator = d0 + d1
    
    # Guard against division by zero
    if abs(denominator) < 1e-12:
        # Fallback to linear interpolation if derivatives cancel out
        # [G] Standard practice to avoid NaN/Inf, though source notes instability
        a0 = v0
        a1 = v1 - v0
        a2 = 0.0
        return a0, a1, a2

    # Calculate scaling factor s
    s = (2.0 * v1 - 2.0 * v0) / denominator
    
    # Scale derivatives
    d0_scaled = s * d0
    d1_scaled = s * d1
    
    # Compute quadratic coefficients
    # p(t) = a0 + a1*t + a2*t^2
    # p(0) = a0 = v0
    # p'(0) = a1 = d0_scaled
    # p'(1) = a1 + 2*a2 = d1_scaled => a2 = (d1_scaled - a1) / 2
    a0 = v0
    a1 = d0_scaled
    a2 = (d1_scaled - a1) / 2.0
    
    return a0, a1, a2
```

**Parameters:**
*   None. The method is fully determined by the input data.

**Edge Cases and Pitfalls:**
*   **Instability:** The solution becomes unstable if $d_0 + d_1 \approx 0$ [S1]. This occurs when the derivatives cancel each other out. The source notes that for the indicator function in the PSR code, this is unlikely because the gradient is steep and derivatives are unlikely to have opposite signs [S1].
*   **Degenerate Input:** If $d_0 + d_1 = 0$, the scaling factor is undefined. A fallback to linear interpolation or a different method (like [Third Order Elimination (Offset)](#third-order-elimination-offset)) is required [S1].

## Subtopics

### Third Order Elimination (Offset)
A quadratic method obtained by adding an offset $o$ to derivatives to eliminate the third order coefficient $a_3$ [S1]. It solves the same problem as Scaling but uses an additive degree of freedom instead of multiplicative scaling [S1].
*   **Definition:** Adds offset $o$ to derivatives $d_0, d_1$ such that $a_3=0$ [S1].
*   **Formula:** $o = \frac{2v_1 - 2v_0 - d_0 - d_1}{2}$ [S1].
*   **Comparison:** This solution is equivalent to the Least-Squares Derivatives method (equation 11 in [S1]) [S1]. It avoids the division instability of Scaling when $d_0 + d_1 \approx 0$ [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per interpolation query. Requires a few arithmetic operations (addition, multiplication, division).
*   **Memory Complexity:** $O(1)$ auxiliary memory.
*   **Strengths:** Produces a quadratic polynomial that strictly interpolates values and scaled derivatives. It is computationally cheaper than full cubic Hermite interpolation [S1].
*   **Limitations:** Unstable when $d_0 + d_1 \approx 0$ [S1]. The scaling can distort the magnitude of the derivatives if the value difference is small relative to the derivative sum [S1].
*   **When to use:** When a quadratic approximation is sufficient and the derivatives are expected to have the same sign (e.g., interpolating indicator functions with steep gradients) [S1].
*   **When not to use:** When derivatives may cancel out ($d_0 \approx -d_1$), in which case [Third Order Elimination (Offset)](#third-order-elimination-offset) or [Least-Squares Derivatives](9450-hermite-interpolation-of-scalar-fields.md) is preferred [S1].
## Background
The method was introduced in the context of accurate isosurface interpolation with Hermite data [S1]. It is part of a family of methods designed to improve upon linear interpolation by incorporating derivative information while managing polynomial degree [S1]. It is implemented in the PSR (Point-Surface Representation) code for interpolating the indicator function [S1].
## Sources
[S1] Accurate Isosurface Interpolation with Hermite Data, https://ieeexplore.ieee.org/document/7335492/, fetched 2026-11-05
