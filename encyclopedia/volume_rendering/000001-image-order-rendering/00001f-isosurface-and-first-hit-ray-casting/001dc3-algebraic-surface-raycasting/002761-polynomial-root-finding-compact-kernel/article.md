# Polynomial Root Finding (Compact Kernel)
## Overview
Polynomial Root Finding (Compact Kernel) is a ray tracing algorithm for locating isosurfaces defined by point primitives using the compact polynomial kernel, where the field function along a ray is a piecewise polynomial [S1]. It distinguishes itself by leveraging the specific algebraic structure of compact kernels to provide robust root-finding strategies, including closed-form solutions for low degrees and Bézier clipping for higher degrees [S1].
## How it works
**Inputs and Outputs:**
*   Input: A ray $r(t) = o + td$, and the coefficients of the compact polynomial kernel field function along the ray.
*   Output: The smallest positive root $t$ corresponding to the first intersection with the isosurface, or a rejection if no root exists in the valid interval.

**Data Structures:**
*   Polynomial Coefficients: A vector $c = [c_0, c_1, \dots, c_n]$ representing the univariate polynomial $P(t) = \sum_{i=0}^n c_i t^i$.
*   Interval: A bounding interval $[t_{min}, t_{max}]$ derived from the primitive support intersection.

**Steps:**
1.  **Determine Degree:** Identify the degree $n$ of the polynomial approximation of the field function along the ray.
2.  **Select Solver:**
    *   If $n \le 4$: Compute the root using a closed-form expression [S1].
    *   If $n > 4$: Use the Bézier clipping algorithm to iteratively converge toward the first root or reject root-free intervals [S1].
3.  **Refine Root:** If a root $t^*$ is found, verify it lies within the valid interval and corresponds to the isovalue.

**Formulas:**
*   Polynomial evaluation: $P(t) = \sum_{i=0}^n c_i t^i$.
*   Bézier Clipping: Iteratively refines the interval containing the root by clipping the Bézier control polygon against the line $y=0$ [S1].

**Parameters:**
*   Degree threshold: $n=4$ separates closed-form from iterative methods [S1].
*   Convergence tolerance: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Closed-form expressions for degrees 3 and 4 may suffer from numerical instability; however, the sources do not specify mitigation strategies beyond using Bézier clipping for $n>4$ [S1].
*   **Boundaries:** The interval must be uniformly subdivided if polynomial approximations are built from Hermite data, as in convolution surfaces [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def find_root_compact_kernel(coefficients, t_min, t_max):
    """
    Finds the first root of the polynomial defined by coefficients
    within the interval [t_min, t_max].
    """
    n = len(coefficients) - 1
    
    if n <= 4:
        # Use closed-form expression for degrees up to 4
        # Implementation details for specific degrees (1-4) are not provided in sources
        # [G] Standard closed-form solvers for linear, quadratic, cubic, and quartic equations
        root = solve_closed_form(coefficients)
    else:
        # Use Bézier clipping for higher degrees
        # [G] Convert polynomial coefficients to Bézier control points
        control_points = polynomial_to_bezier(coefficients)
        root = bezier_clipping(control_points, t_min, t_max)
        
    return root

def bezier_clipping(control_points, t_min, t_max):
    """
    Iteratively clips the Bézier curve to find the root.
    """
    # [G] Standard Bézier clipping algorithm
    # 1. Evaluate the curve and its derivative at the midpoint
    # 2. Clip the control polygon against the tangent line
    # 3. Update the interval
    # 4. Repeat until convergence or interval is too small
    pass
```
## Subtopics

### Kernel Root Finding
Kernel Root Finding is a variant that handles specific low-degree cases ($n=1, 2$) with closed-form expressions and higher degrees ($n>2$) with a Newton-style approach [S2]. It is chosen when differentiability and robust handling of singularities (vanishing leading coefficients) are required [S2]. For $n=1$ and $n=2$, it uses a robust quadratic solver following Kahan [2004] [S2]. For $n>2$, it uses a fast and accurate Newton-style approach [Yuksel 2022] [S2]. A key pitfall is the handling of vanishing leading coefficients $c_n$; instead of branching, it returns a root with value $\infty$, and subsequent operations handle roots at infinity by returning limit values [S2]. If the quadratic factor in the denominator of weight derivatives overflows, the contribution of the root to the derivative is set to zero [S2].

## Operating characteristics
*   **Time Complexity:** $O(1)$ for degrees $\le 4$ due to closed-form solutions; $O(k)$ for degrees $>4$ where $k$ is the number of Bézier clipping iterations [S1].
*   **Memory Complexity:** $O(n)$ to store polynomial coefficients [S1].
*   **Strengths:** Robust handling of low-degree polynomials; efficient for high-degree polynomials via Bézier clipping [S1].
*   **Limitations:** Closed-form expressions may be numerically unstable for higher degrees (mitigated by using Bézier clipping for $n>4$) [S1].
*   **When to use:** When rendering point primitives defined with compact polynomial kernels [S1].
*   **When not to use:** When the field function is not piecewise polynomial or when higher precision is required for degrees 3 and 4 (consider Newton-style for all $n>2$ as in Kernel Root Finding) [S2].
## Background
The method originates from ray tracing algorithms for scale-invariant integral surfaces, where polynomial root finding is used to locate isosurfaces along rays [S1]. It builds on the property that the field function along a ray for compact polynomial kernels is a piecewise polynomial [S1]. The Bézier clipping algorithm was introduced by [NN94] and [KSN08] for this purpose [S1].
## Sources
[S1] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
[S2] Moment Bounds are Differentiable: Efficiently Approximating Measures in Inverse Rendering, https://doi.org/10.1145/3730899, fetched 2026-10-27
