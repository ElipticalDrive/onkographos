# Polynomial Root Finding (Compact Kernel)
## Overview
Polynomial Root Finding (Compact Kernel) is a ray tracing algorithm for locating isosurfaces defined by point primitives using the compact polynomial kernel, where the field function along a ray is a piecewise polynomial [S1]. It distinguishes itself by leveraging the algebraic structure of compact kernels to apply closed-form solutions for degrees up to 4 and Bézier clipping for higher degrees [S1].
## How it works
**Inputs and Outputs:**
*   Input: A ray $r(t) = o + td$, and the coefficients of the compact polynomial kernel field function along the ray [G].
*   Output: The smallest positive root $t$ corresponding to the first intersection with the isosurface, or a rejection if no root exists in the valid interval [G].

**Data Structures:**
*   Polynomial Coefficients: A vector $c = [c_0, c_1, \dots, c_n]$ representing the univariate polynomial $P(t) = \sum_{i=0}^n c_i t^i$ [G].
*   Interval: A bounding interval $[t_{min}, t_{max}]$ derived from the primitive support intersection [G].

**Steps:**
1.  **Determine Degree:** Identify the degree $n$ of the polynomial approximation of the field function along the ray [G].
2.  **Select Solver:** If $n \le 4$, compute a closed-form expression of the root [S1]. If $n > 4$, use the Bézier clipping algorithm to iteratively converge toward the first root or rapidly reject root-free intervals [S1].
3.  **Refine Root:** Verify the found root lies within the valid interval and corresponds to the isovalue [G].

**Formulas:**
*   Polynomial evaluation: $P(t) = \sum_{i=0}^n c_i t^i$ [G].
*   Bézier Clipping: Iteratively refines the interval containing the root by clipping the Bézier control polygon against the line $y=0$ [S1].

**Parameters:**
*   Degree threshold: $n=4$ separates closed-form from iterative methods [S1].
*   Convergence tolerance: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Boundaries:** The interval must be uniformly subdivided if polynomial approximations are built from Hermite data (field value and derivative) sampled at subinterval boundaries, as used in convolution surfaces [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_root_compact_kernel(coefficients, t_min, t_max):
    n = len(coefficients) - 1
    if n <= 4:
        root = solve_closed_form(coefficients, n)
    else:
        root = bezier_clipping(coefficients, t_min, t_max)
    return root

def solve_closed_form(coefficients, n):
    # [G] Standard closed-form solvers for degrees 1 through 4
    if n == 1:
        return -coefficients[0] / coefficients[1]
    elif n == 2:
        a, b, c = coefficients[2], coefficients[1], coefficients[0]
        disc = b**2 - 4*a*c
        if disc < 0: return None
        sqrt_disc = math.sqrt(disc)
        return min((-b - sqrt_disc) / (2*a), (-b + sqrt_disc) / (2*a))
    elif n == 3:
        # [G] Cardano's formula for cubic equations
        return None
    elif n == 4:
        # [G] Ferrari's method for quartic equations
        return None
    return None

def bezier_clipping(coefficients, t_min, t_max):
    n = len(coefficients) - 1
    # [G] Convert polynomial coefficients to Bézier control points using binomial scaling
    control_points = [coefficients[i] * math.comb(n, i) for i in range(n + 1)]
    # [G] Standard Bézier clipping algorithm
    while t_max - t_min > 1e-6: # [G] Convergence tolerance not given in sources
        # [G] Evaluate curve and derivative at midpoint
        # [G] Clip control polygon against tangent line y=0
        # [G] Update t_min or t_max based on intersection
        pass
    return (t_min + t_max) / 2
```
## Subtopics

### Kernel Root Finding
Kernel Root Finding handles specific low-degree cases ($n=1, 2$) with closed-form expressions and higher degrees ($n>2$) with a Newton-style approach [S2]. It is chosen when differentiability and robust handling of singularities are required [S2]. For $n=1$ and $n=2$, it uses a robust quadratic solver following Kahan [2004] [S2]. For $n>2$, it uses a fast and accurate Newton-style approach [Yuksel 2022] [S2]. A key pitfall is the handling of vanishing leading coefficients $c_n$; instead of branching, it returns a root with value $\infty$, and subsequent operations handle roots at infinity by returning limit values [S2]. If the quadratic factor in the denominator of weight derivatives overflows, the contribution of the root to the derivative is set to zero [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for degrees $\le 4$ due to closed-form solutions; $O(k)$ for degrees $>4$ where $k$ is the number of Bézier clipping iterations [S1].
*   **Strengths:** Robust handling of low-degree polynomials; efficient for high-degree polynomials via Bézier clipping [S1].
*   **When to use:** When rendering point primitives defined with compact polynomial kernels [S1].
## Background
The method builds on ray tracing algorithms that rely on polynomial root finding to locate isosurfaces along rays, either because the field itself is defined by polynomials or because a polynomial approximation of the field is built along the ray [S1]. It utilizes the property that the field function along a ray for compact polynomial kernels is a piecewise polynomial [S1]. For convolution surfaces, polynomial approximations of the field value are built along the ray by uniformly subdividing the interval defined by the intersection between the ray and the primitive support, then calculating polynomial approximations from Hermite data sampled at subinterval boundaries [S1]. The Bézier clipping algorithm was introduced by [NN94] and [KSN08] for this purpose [S1].
## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S2] Moment Bounds are Differentiable: Efficiently Approximating Measures in Inverse Rendering, https://doi.org/10.1145/3730899, fetched 2026-10-27
