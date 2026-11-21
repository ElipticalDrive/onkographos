# de Casteljau-based root finding

Also called: Root Finding: de Casteljau Subdivision (LR81), de Casteljau Subdivision

## Overview
This method finds the intersection of a viewing ray with an isosurface defined by a trivariate polynomial in Bernstein form by iteratively subdividing the control polygon using the de Casteljau algorithm at the intersection with the zero axis [S1]. It distinguishes itself from sibling methods like [Neubauer Iterative Root Finder](2422-neubauer-iterative-root-finder.md) and [Knot insertion root finding](#knot-insertion-root-finding) by offering quadratic convergence speed and constant memory usage without index operations, making it significantly faster and more suitable for GPU architectures than linear-convergence knot insertion approaches [S1].

## How it works
**Inputs:**
*   Polynomial coefficients in Bernstein basis defining the distance field or implicit function $f(t)$ along the ray parameter $t \in [0, 1]$ [S1].
*   Isosurface value $c$ (typically 0 for normalized distance fields) [S1].
*   Maximum number of iterations $N_{iter}$ (typically 3 to 7) [S1].

**Outputs:**
*   The smallest root $t^* \in [0, 1]$ such that $f(t^*) - c = 0$ [S1].

**Data Structures:**
The algorithm operates on the control polygon of the Bernstein polynomial [S1]. For a polynomial of degree $d$, the control polygon consists of $d+1$ control points $P_i$ [S1]. In the context of volume rendering, the polynomial is often univariate along the ray, derived from the trivariate field [S1]. The values are represented as scalars $b_i$ corresponding to the Bernstein coefficients [S1].

**Steps:**
1.  **Initialization:** Let the current control points be $P_i^{(0)} = b_i$ for $i = 0, \dots, d$. Let the current parameter interval be $[t_{min}, t_{max}] = [0, 1]$ [S1].
2.  **Iteration Loop:** Repeat for $k = 1$ to $N_{iter}$:
    a.  **Find Intersection:** Identify the index $j$ where the control polygon crosses the zero axis (or the value $c$) [S1]. Specifically, find $j$ such that $P_j^{(k-1)} - c$ and $P_{j+1}^{(k-1)} - c$ have opposite signs or one is zero [S1].
    b.  **Compute Subdivision Parameter:** Calculate $t_{sub} = \frac{P_j^{(k-1)} - c}{P_j^{(k-1)} - P_{j+1}^{(k-1)}}$ using linear interpolation at the crossing segment [S1].
    c.  **De Casteljau Subdivision:** Perform the de Casteljau algorithm to split the control polygon at $t_{sub}$ [S1]. This generates two new sets of control points: left polygon $P^{(L)}$ and right polygon $P^{(R)}$ [S1].
    d.  **Selection:** Select the sub-polygon ($P^{(L)}$ or $P^{(R)}$) that contains the root by checking the signs of the control points relative to $c$ [S1].
    e.  **Update:** Set the current control points to the selected sub-polygon and update $[t_{min}, t_{max}]$ to the corresponding sub-interval [S1].
3.  **Result:** The root is approximated by the midpoint of the final interval $[t_{min}, t_{max}]$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def de_casteljau_root_finding(bernstein_coeffs, target_value, max_iterations):
    """
    Finds the root of a polynomial in Bernstein form using de Casteljau subdivision.
    
    Args:
        bernstein_coeffs: List of float coefficients b_0, ..., b_d.
        target_value: The isosurface value c (e.g., 0).
        max_iterations: Integer, typically 3-7.
        
    Returns:
        float: Approximate root t in [0, 1].
    """
    # Current control points (constant memory for fixed degree d)
    p = list(bernstein_coeffs)
    degree = len(p) - 1
    
    # Interval bounds
    t_min = 0.0
    t_max = 1.0
    
    # Pre-allocate buffers to satisfy constant memory claim
    left = [0.0] * (degree + 1)
    right = [0.0] * (degree + 1)
    
    for _ in range(max_iterations):
        # 1. Find the segment crossing the target value
        crossing_index = -1
        for i in range(degree):
            val_i = p[i] - target_value
            val_next = p[i+1] - target_value
            if val_i * val_next <= 0:
                crossing_index = i
                break
        
        # Fallback if no crossing found (root at boundary or degenerate)
        if crossing_index == -1:
            return (t_min + t_max) / 2.0
            
        # 2. Calculate subdivision parameter t_sub at the intersection
        v1 = p[crossing_index] - target_value
        v2 = p[crossing_index+1] - target_value
        denom = v1 - v2
        t_sub = v1 / denom if abs(denom) > 1e-12 else 0.5
            
        # 3. Perform de Casteljau subdivision at t_sub
        # Compute Left Polygon P_L (for interval [0, t_sub])
        left[0] = p[0]
        for j in range(1, degree + 1):
            left[j] = left[j-1] * (1.0 - t_sub) + p[j] * t_sub
            
        # Compute Right Polygon P_R (for interval [t_sub, 1])
        right[degree] = p[degree]
        for j in range(degree - 1, -1, -1):
            right[j] = right[j+1] * t_sub + p[j] * (1.0 - t_sub)
            
        # 4. Determine which side contains the root
        # Check sign change in the left polygon relative to target
        val_left_start = left[0] - target_value
        val_left_end = left[degree] - target_value  # Value at t_sub
        
        if val_left_start * val_left_end <= 0:
            # Root is in the left polygon
            for j in range(degree + 1):
                p[j] = left[j]
            t_max = t_sub
        else:
            # Root is in the right polygon
            for j in range(degree + 1):
                p[j] = right[j]
            t_min = t_sub
            
    # Return the midpoint of the final interval
    return (t_min + t_max) / 2.0
```

**Parameters:**
*   `max_iterations`: 3 to 7, depending on the interpolation type [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors can cause the control polygon to fail to bracket the root correctly [S1].
*   **Boundaries:** If the root is exactly at 0 or 1, the sign check might fail if not handled carefully [S1].
*   **Multiple Roots:** The method assumes an isolated root; if multiple roots exist, the control polygon may not cross the zero axis in a simple manner [S1].
*   **CUDA Divergence:** While LR81 itself is well-suited for CUDA, combined methods using Regula Falsi for isolation may suffer from diverging branches [S1].

## Subtopics

### Knot insertion root finding
A root finding algorithm based on knot insertion. It has linear convergence speed and linear complexity per step. It requires index operations and memory usage grows with every iteration step, limiting the maximum number of iterations. For degree 9 (tri-cubic), it has an execution time of 1.0 (relative), MSE 0.034, and max error 0.85. The MR07 knot insertion variant is more than 10 times slower than LR81 [S1].

### Regula Falsi with De Casteljau
A root finding method using Regula Falsi combined with De Casteljau subdivision for root isolation. Although the second phase (Regula Falsi) outperforms LR81 when a root is isolated, the isolation step is not well suited for CUDA due to diverging branches [S1].

## Operating characteristics
*   **Time Complexity:** Quadratic complexity per step due to the de Casteljau algorithm [S1].
*   **Convergence:** Quadratic convergence speed [S1].
*   **Memory Complexity:** Constant memory usage. It can be implemented without index operations [S1].
*   **Performance:** For degree 9 (tri-cubic), execution time is 0.076 (relative to worst result 1.0), MSE 0.003, max error 0.66 [S1]. It is more than 10 times faster than the MR07 (knot insertion) algorithm [S1].
*   **Strengths:** Fast, constant memory, no index operations, suitable for GPU (CUDA) [S1].
*   **Limitations:** Requires a fixed number of iterations (3-7) chosen based on interpolation type; combined isolation methods suffer from CUDA branch divergence [S1].

## Background
The method is based on the idea of subdividing the control polygon using the de Casteljau algorithm at the intersection of the control polygon with the zero axis. It is referenced as [LR81] in the literature. It is used in volume rendering for finding the first intersection of a viewing ray with an isosurface defined by trivariate polynomials in Bernstein form [S1].

## Sources
[S1] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
