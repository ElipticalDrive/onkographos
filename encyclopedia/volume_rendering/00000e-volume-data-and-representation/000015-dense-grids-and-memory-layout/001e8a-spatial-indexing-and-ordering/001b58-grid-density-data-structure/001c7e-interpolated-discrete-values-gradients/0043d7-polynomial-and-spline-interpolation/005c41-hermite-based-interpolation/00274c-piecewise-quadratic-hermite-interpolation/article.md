# Piecewise Quadratic Hermite Interpolation

Also called: Piecewise Quadratic Bézier Hermite Interpolation, Quadratic Polynomial Interpolation for SCALIS

## Overview
Piecewise Quadratic Hermite Interpolation is a method for interpolating a normalized scalar field within a 1D interval using two connected quadratic Bézier curves, ensuring $C^1$ continuity while respecting Hermite data (value and derivative) at both endpoints. It solves the incompatibility of single quadratic interpolation with four constraints (two values, two derivatives) by splitting the interval, providing a numerically stable and efficient alternative to cubic interpolation for root isolation in volume rendering [S1]. It is specifically designed for use within the SCALIS framework, where it operates on a normalized field $g \circ f \circ r^{-1}$ to improve convergence for small numbers of subdivisions compared to cubic interpolation on the original field [S1].

## How it works
**Inputs:**
*   Interval endpoints $t_{begin}$ and $t_{end}$ with length $L = t_{end} - t_{begin}$.
*   Hermite data at endpoints: $(h_1, h'_1)$ at $t_{begin}$ and $(h_2, h'_2)$ at $t_{end}$, where $h$ is the normalized field value and $h'$ is the derivative with respect to the normalized coordinate.

**Outputs:**
*   A continuous, piecewise quadratic function defined by six control points $Q_0, \dots, Q_5$.
*   The interpolated value and derivative at any query point $t \in [t_{begin}, t_{end}]$.

**Data Structures:**
*   Control Points: An array of 6 values $Q_i$ representing the ordinates of the Bézier control polygon. The abscissae are fixed and equally spaced.
    *   $Q_0, Q_1, Q_2$ define the first quadratic Bézier curve on $[t_{begin}, t_{mid}]$.
    *   $Q_3, Q_4, Q_5$ define the second quadratic Bézier curve on $[t_{mid}, t_{end}]$.
    *   $t_{mid} = (t_{begin} + t_{end}) / 2$.

**Steps:**
1.  **Split Interval:** Divide the initial interval $[t_{begin}, t_{end}]$ into two equal sub-intervals: $[t_{begin}, t_{mid}]$ and $[t_{mid}, t_{end}]$.
2.  **Set Endpoint Constraints:**
    *   Set $Q_0 = h_1$ (value at $t_{begin}$).
    *   Set $Q_5 = h_2$ (value at $t_{end}$).
    *   The derivatives $h'_1$ and $h'_2$ constrain the tangents of the Bézier curves. For a quadratic Bézier curve with control points $P_0, P_1, P_2$ on interval length $\Delta t$, the derivative at $P_0$ is $\frac{2}{\Delta t}(P_1 - P_0)$.
3.  **Determine Intermediate Control Points:**
    *   The control points $Q_1$ and $Q_4$ are determined by the endpoint derivatives.
        *   $Q_1 = Q_0 + \frac{L}{4} h'_1$ (since $\Delta t = L/2$, derivative factor is $2/(L/2) = 4/L$, so $Q_1 - Q_0 = \frac{L}{4} h'_1$).
        *   $Q_4 = Q_5 - \frac{L}{4} h'_2$ (derivative at end is $\frac{2}{L/2}(Q_5 - Q_4) = h'_2 \implies Q_5 - Q_4 = \frac{L}{4} h'_2$).
    *   The shared control point $Q_2 = Q_3$ is calculated to ensure continuity and smoothness. It lies on the line segment connecting $Q_1$ and $Q_4$.
    *   The ordinate for $Q_2$ (and $Q_3$) is given by:
        $$ Q_2 = Q_3 = \frac{1}{2}(Q_1 + Q_4) + \frac{L}{8}(h'_1 - h'_2) $$
        Substituting $Q_1$ and $Q_4$:
        $$ Q_2 = \frac{1}{2}(h_1 + h_2) + \frac{L}{8}(h'_1 - h'_2) $$
        [S1]
4.  **Evaluation:** To evaluate the function at a query point $t$:
    *   If $t \in [t_{begin}, t_{mid}]$, use the first Bézier curve defined by $Q_0, Q_1, Q_2$.
    *   If $t \in [t_{mid}, t_{end}]$, use the second Bézier curve defined by $Q_3, Q_4, Q_5$.
    *   The quadratic Bézier basis functions for parameter $u \in [0, 1]$ are $B_0(u) = (1-u)^2$, $B_1(u) = 2u(1-u)$, $B_2(u) = u^2$. Map $t$ to $u$ within the respective sub-interval.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def piecewise_quadratic_hermite_interpolate(t, t_begin, t_end, h1, h1_prime, h2, h2_prime):
    """
    Interpolate value at t using piecewise quadratic Hermite interpolation.
    
    Args:
        t: Query point in [t_begin, t_end]
        t_begin: Start of interval
        t_end: End of interval
        h1: Value at t_begin
        h1_prime: Derivative at t_begin
        h2: Value at t_end
        h2_prime: Derivative at t_end
        
    Returns:
        Interpolated value at t
    """
    L = t_end - t_begin
    t_mid = t_begin + L / 2.0
    
    # Calculate control points
    Q0 = h1
    Q5 = h2
    
    # Q1 and Q4 derived from derivatives
    # Derivative of quadratic Bezier P0,P1,P2 at P0 is 2*(P1-P0)/dt
    # Here dt = L/2, so 2/(L/2) = 4/L
    # h1_prime = 4/L * (Q1 - Q0) => Q1 = Q0 + L/4 * h1_prime
    Q1 = Q0 + (L / 4.0) * h1_prime
    
    # Derivative at P2 is 2*(P2-P1)/dt
    # h2_prime = 4/L * (Q5 - Q4) => Q4 = Q5 - L/4 * h2_prime
    Q4 = Q5 - (L / 4.0) * h2_prime
    
    # Shared control point Q2 = Q3
    # Formula from source: 1/2*(h1 + h2) + L/8*(h1_prime - h2_prime)
    # Note: The source formula uses h1, h2 directly. 
    # Let's verify if this matches the geometric derivation from Q1, Q4.
    # Geometric: Q2 lies on line Q1-Q4. 
    # The specific formula provided in [S1] Eq 20 is:
    Q2 = 0.5 * (h1 + h2) + (L / 8.0) * (h1_prime - h2_prime)
    Q3 = Q2
    
    # Determine which sub-interval t falls into
    if t <= t_mid:
        # First interval [t_begin, t_mid]
        # Map t to u in [0, 1]
        u = (t - t_begin) / (t_mid - t_begin)
        # Quadratic Bezier evaluation
        # B0 = (1-u)^2, B1 = 2u(1-u), B2 = u^2
        val = (1 - u)**2 * Q0 + 2 * u * (1 - u) * Q1 + u**2 * Q2
    else:
        # Second interval [t_mid, t_end]
        # Map t to u in [0, 1]
        u = (t - t_mid) / (t_end - t_mid)
        # Quadratic Bezier evaluation with Q3, Q4, Q5
        val = (1 - u)**2 * Q3 + 2 * u * (1 - u) * Q4 + u**2 * Q5
        
    return val
```

**Edge Cases and Pitfalls:**
*   **Incompatible Hermite Data:** A single quadratic polynomial cannot satisfy four arbitrary constraints (value and derivative at two points). This method avoids this by using two pieces [S1].
*   **Missed Isovalues:** Polynomial interpolation may underestimate variations in the normalized field, potentially missing isosurface crossings. This is addressed by using a Rational Bézier Oracle (see Subtopics) [S1].
*   **Precision:** The method is noted to be numerically stable and more efficient than cubic interpolation [S1].

## Subtopics

### Rational Bézier Oracle for Root Isolation
A heuristic used when polynomial interpolation might overestimate normalized field values (e.g., large radius variations), potentially missing the isosurface. It uses rational Bézier curves to ensure the interpolation is arbitrarily closer to the control polygon, increasing the chance of detecting an isovalue crossing until a root is isolated [S1].
*   **Definition:** Uses rational Bézier curves instead of polynomial curves for interpolation.
*   **Steps:**
    1.  Use rational Bézier curves for interpolation.
    2.  Set the weight of the middle control point to 3.
    3.  Compute roots using the same method as for polynomials (since poles are not in the range of interest).
    4.  Continue using rational curves until a sign change exists between interval endpoints (root isolated) [S1].
*   **Pitfall:** Polynomial interpolation can present smaller variations than the actual field function, resulting in possible missed isovalue crossings. Rational Bézier curves address this by being closer to the control polygon [S1].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per evaluation. The calculation involves a fixed number of arithmetic operations.
*   **Memory Complexity:** $O(1)$ auxiliary memory for control points.
*   **Performance:** More efficient and numerically stable than cubic interpolation [S1]. For Cauchy kernels, it yields much better interpolations for small numbers of subdivisions compared to cubic interpolation on the original field [S1].
*   **Strengths:** Exact interpolation in specific cases due to field normalization; simpler root computation; handles incompatible Hermite data gracefully [S1].
*   **Limitations:** Requires splitting the interval; may miss isosurface crossings without the rational Bézier heuristic [S1].
*   **When to use:** When high efficiency and stability are required for root isolation in volume rendering, particularly with compact polynomial kernels or when subdivision count is limited [S1].
*   **When not to use:** When $C^2$ continuity is required (this method provides $C^1$) [S1].

## Background
This method originates from the SCALIS (Scale-Invariant Integral Surfaces) framework, presented in the paper "Fast Ray Tracing of Scale-Invariant Integral Surfaces" [S1]. It was developed to address the limitations of cubic interpolation in the context of scale-invariant surface rendering, offering a balance between accuracy and computational cost. The method leverages the normalization of the field to achieve exact interpolation in certain configurations [S1].

## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
