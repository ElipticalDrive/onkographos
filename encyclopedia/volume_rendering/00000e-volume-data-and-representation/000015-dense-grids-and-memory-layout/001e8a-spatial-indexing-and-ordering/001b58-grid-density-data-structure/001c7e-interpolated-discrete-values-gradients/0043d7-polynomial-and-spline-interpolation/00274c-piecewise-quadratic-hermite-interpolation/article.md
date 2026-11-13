# Piecewise Quadratic Hermite Interpolation

Also called: Piecewise Quadratic Bézier Hermite Interpolation, Quadratic Polynomial Interpolation for SCALIS

## Overview
Piecewise Quadratic Hermite Interpolation is a method for interpolating a normalized scalar field within a grid interval using two connected quadratic Bézier curves, respecting both value and derivative (Hermite) data at the interval endpoints. It solves the incompatibility of single quadratic interpolation with four constraints (two values, two derivatives) by splitting the interval, providing a numerically stable and efficient alternative to cubic interpolation while maintaining $C^1$ continuity [S1]. It is distinct from [Hermite Interpolation of Scalar Fields](9450-hermite-interpolation-of-scalar-fields.md) (which typically uses cubic polynomials) and [B-spline Field Interpolation](11066-b-spline-field-interpolation.md) (which uses global basis functions) by employing local piecewise quadratic Bézier segments [S1].

## How it works
**Inputs:**
- Interval endpoints $t_{begin}$ and $t_{end}$ with length $L = t_{end} - t_{begin}$.
- Hermite data at endpoints: $(h_1, h'_1)$ at $t_{begin}$ and $(h_2, h'_2)$ at $t_{end}$, where $h$ is the normalized field value and $h'$ is the normalized derivative [S1].

**Outputs:**
- A continuous, piecewise quadratic function $u(t)$ defined on $[t_{begin}, t_{end}]$.
- Control points for the two Bézier curves.

**Data Structures:**
- Six control points $Q_0, \dots, Q_5$ representing the ordinates of two quadratic Bézier curves.
- The abscissae (x-coordinates) of control points are fixed: equally spaced. For the first curve on $[t_{begin}, t_{mid}]$ and second on $[t_{mid}, t_{end}]$ where $t_{mid} = (t_{begin} + t_{end})/2$, the control points are:
  - Curve 1: $Q_0$ at $t_{begin}$, $Q_1$ at $t_{mid}$, $Q_2$ at $t_{mid}$ (shared).
  - Curve 2: $Q_3$ at $t_{mid}$ (shared), $Q_4$ at $t_{mid}$, $Q_5$ at $t_{end}$.
  - Note: The source specifies control points are equally spaced in abscissa. For quadratic Bézier, the middle control point's abscissa is the midpoint of the segment. Thus, $Q_1$ and $Q_2$ (and $Q_3, Q_4$) share the midpoint abscissa $t_{mid}$ [S1].

**Steps:**
1.  **Split Interval:** Divide the interval $[t_{begin}, t_{end}]$ into two equal sub-intervals: $[t_{begin}, t_{mid}]$ and $[t_{mid}, t_{end}]$, where $t_{mid} = \frac{t_{begin} + t_{end}}{2}$ [S1].
2.  **Set Endpoint Control Points:**
    -   Set $Q_0 = h_1$ (value at $t_{begin}$).
    -   Set $Q_5 = h_2$ (value at $t_{end}$).
    -   The derivatives $h'_1$ and $h'_2$ constrain the positions of $Q_1$ and $Q_4$ relative to $Q_0$ and $Q_5$ respectively in the context of the Bézier derivative formula, but the source explicitly defines the ordinates of $Q_0, Q_1$ and $Q_4, Q_5$ to satisfy Hermite data. Specifically, for a quadratic Bézier curve $B(t)$ on $[0,1]$ with control points $P_0, P_1, P_2$, $B'(0) = 2(P_1 - P_0)$ and $B'(1) = 2(P_2 - P_1)$.
    -   For the first sub-interval of length $L/2$:
        -   $Q_0 = h_1$.
        -   $Q_1$ is determined by $h'_1$. $B'_1(0) = h'_1 \implies 2(Q_1 - Q_0) / (L/2) = h'_1 \implies Q_1 = h_1 + \frac{L}{4}h'_1$.
    -   For the second sub-interval of length $L/2$:
        -   $Q_5 = h_2$.
        -   $Q_4$ is determined by $h'_2$. $B'_2(1) = h'_2 \implies 2(Q_5 - Q_4) / (L/2) = h'_2 \implies Q_4 = h_2 - \frac{L}{4}h'_2$.
    -   *Correction based on source text:* The source states "Set ordinates of $Q_0, Q_1$ (first interval) and $Q_4, Q_5$ (second interval) to satisfy Hermite data." It then gives a formula for $Q_2=Q_3$. It does not explicitly give the formula for $Q_1$ and $Q_4$ in the snippet, but standard Hermite-Bézier conversion applies. However, the source emphasizes the calculation of the *shared* points $Q_2, Q_3$. Let's look at the formula provided: $\frac{1}{2}(h_1 + h_2) + \frac{L}{8}(h'_1 - h'_2)$. This is the value for $Q_2$ and $Q_3$.
3.  **Calculate Shared Control Points:**
    -   Enforce continuity $Q_2 = Q_3$.
    -   Enforce smoothness by lying on the line joining $Q_1$ and $Q_4$. The source states they "should lie on the line joining the two adjacent control points $Q_1$ and $Q_4$". This implies $Q_2$ (and $Q_3$) is the midpoint of $Q_1$ and $Q_4$? No, the formula is specific.
    -   Calculate the ordinate for $Q_2$ and $Q_3$ using Equation (20) [S1]:
        $$ Q_2 = Q_3 = \frac{1}{2}(h_1 + h_2) + \frac{L}{8}(h'_1 - h'_2) $$
    -   The source notes this value is equal to the cubic Hermite interpolation at the midpoint and the average of two Hermite-Birkhoff interpolations [S1].
4.  **Construct Curves:**
    -   Curve 1: Quadratic Bézier with control points $Q_0, Q_1, Q_2$.
    -   Curve 2: Quadratic Bézier with control points $Q_3, Q_4, Q_5$.
    -   Evaluate the field at a query point $t$ by determining which sub-interval it falls in and evaluating the corresponding Bézier curve [S1].

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
        h1: Field value at t_begin
        h1_prime: Field derivative at t_begin
        h2: Field value at t_end
        h2_prime: Field derivative at t_end
        
    Returns:
        Interpolated field value
    """
    L = t_end - t_begin
    t_mid = (t_begin + t_end) / 2.0
    
    # Calculate control points ordinates
    # Q0 and Q5 are the endpoint values
    Q0 = h1
    Q5 = h2
    
    # Q1 and Q4 are derived from derivatives and endpoint values
    # For quadratic Bezier B(t) on [0,1] with control points P0, P1, P2:
    # B'(0) = 2(P1 - P0) / duration
    # B'(1) = 2(P2 - P1) / duration
    # Duration for each sub-interval is L/2
    
    # For first sub-interval [t_begin, t_mid]:
    # h1_prime = 2 * (Q1 - Q0) / (L/2) => Q1 = Q0 + h1_prime * L / 4
    Q1 = Q0 + h1_prime * L / 4.0
    
    # For second sub-interval [t_mid, t_end]:
    # h2_prime = 2 * (Q5 - Q4) / (L/2) => Q4 = Q5 - h2_prime * L / 4
    Q4 = Q5 - h2_prime * L / 4.0
    
    # Q2 and Q3 are the shared middle control points
    # Formula (20) from source
    Q2 = 0.5 * (h1 + h2) + (L / 8.0) * (h1_prime - h2_prime)
    Q3 = Q2  # Continuity Q2 = Q3
    
    # Determine which sub-interval t falls into
    if t < t_mid:
        # Evaluate first quadratic Bezier curve: Q0, Q1, Q2
        # Parameter u in [0, 1] for the sub-interval
        u = (t - t_begin) / (t_mid - t_begin)
        # Quadratic Bezier formula: (1-u)^2 P0 + 2(1-u)u P1 + u^2 P2
        val = (1 - u)**2 * Q0 + 2 * (1 - u) * u * Q1 + u**2 * Q2
    else:
        # Evaluate second quadratic Bezier curve: Q3, Q4, Q5
        # Parameter u in [0, 1] for the sub-interval
        u = (t - t_mid) / (t_end - t_mid)
        # Quadratic Bezier formula: (1-u)^2 P0 + 2(1-u)u P1 + u^2 P2
        # Here P0=Q3, P1=Q4, P2=Q5
        val = (1 - u)**2 * Q3 + 2 * (1 - u) * u * Q4 + u**2 * Q5
        
    return val
```

**Edge Cases and Pitfalls:**
-   **Incompatible Hermite Data:** A single quadratic polynomial cannot satisfy four constraints (two values, two derivatives). This method avoids this by using two quadratics [S1].
-   **Missed Isovalues:** Polynomial interpolation may underestimate field variations, potentially missing isosurface crossings. This is addressed by using a Rational Bézier Oracle (see Subtopics) [S1].
-   **Precision:** The method is noted to be numerically stable and efficient compared to cubic interpolation [S1].

## Subtopics

### Rational Bézier Oracle for Root Isolation
A heuristic used when polynomial interpolation might overestimate normalized field values (e.g., due to large radius variations), potentially missing the isosurface. It uses rational Bézier curves to ensure the interpolation is arbitrarily closer to the control polygon, increasing the chance of detecting an isovalue crossing until a root is isolated [S1].
-   **Definition:** Uses rational Bézier curves instead of polynomial curves for root finding [S1].
-   **Steps:**
    1.  Use rational Bézier curves for interpolation [S1].
    2.  Set the weight of the middle control point to 3 [S1].
    3.  Compute roots using the same method as for polynomials (since poles are not in the range of interest) [S1].
    4.  Continue using rational curves until a sign change exists between interval endpoints (root isolated) [S1].
-   **Pitfall:** Polynomial interpolation can present smaller variations than the actual field function, resulting in possible missed isovalue crossings. Rational Bézier curves address this by being closer to the control polygon [S1].

## Operating characteristics
-   **Time Complexity:** $O(1)$ per query point, involving basic arithmetic operations. Root finding requires iterative subdivision or oracle calls, but the quadratic nature allows for efficient closed-form or simple iterative solutions compared to cubic [S1].
-   **Memory Complexity:** $O(1)$ auxiliary memory per interval for control points.
-   **Performance:** More efficient and numerically stable than cubic interpolation on the original field. For Cauchy kernels, it yields much better interpolations for small numbers of subdivisions compared to cubic interpolation [S1].
-   **Strengths:** Exact interpolation in specific cases due to field normalization; handles Hermite data at both endpoints without ignoring derivatives; simple and stable root computation [S1].
-   **Limitations:** Requires derivative information (Hermite data); potential for missed isosurfaces if not combined with the Rational Bézier Oracle [S1].
-   **When to use:** When efficient, stable interpolation of normalized fields with derivative information is required, particularly in ray tracing of scale-invariant integral surfaces [S1].
-   **When not to use:** When higher-order continuity ($C^2$) is strictly required and cannot be approximated, or when derivative data is unavailable.

## Background
Originated in the context of ray tracing scale-invariant integral surfaces, specifically in the SCALIS framework. The method was proposed to address the limitations of cubic interpolation in terms of efficiency and stability, while maintaining the ability to respect Hermite data at interval endpoints [S1]. It is part of the broader effort to optimize volume rendering and isosurface extraction by leveraging field normalization and efficient polynomial interpolation [S1].

## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
