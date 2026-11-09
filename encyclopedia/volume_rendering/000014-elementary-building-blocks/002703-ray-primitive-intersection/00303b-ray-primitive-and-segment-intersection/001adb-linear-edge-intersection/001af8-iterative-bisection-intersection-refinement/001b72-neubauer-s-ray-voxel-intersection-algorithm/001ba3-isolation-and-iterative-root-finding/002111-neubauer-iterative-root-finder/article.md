# Neubauer Iterative Root Finder

Also called: Neubauer's method

## Overview
The Neubauer Iterative Root Finder locates the first positive root of the ray-volume intersection polynomial within a voxel cell [S1][S2, unofficial]. It is distinguished from analytical solvers like Schwarze’s cubic solver by its iterative re-parameterization into sub-intervals, offering slightly higher performance and robust single-precision numerical stability [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Cubic polynomial coefficients $a, b, c, d$ for $P(t) = at^3 + bt^2 + ct + d$, and an initial search interval $[t_{min}, t_{max}]$ corresponding to the ray's entry and exit parameters for the voxel cell [S1][S2, unofficial].
*   **Output:** The parameter $t$ along the ray corresponding to the first positive root (intersection point) [S1][S2, unofficial].

**Data Structures:**
The method operates on standard floating-point variables for ray parameters and polynomial coefficients [G]. No complex data structures are required [G].

**Algorithm Steps:**
1. Initialize the search interval $[t_{min}, t_{max}]$ to the ray's entry and exit parameters for the voxel cell [G].
2. Iteratively re-parameterize the ray into smaller sub-intervals within the current interval [S1][S2, unofficial].
3. Evaluate the polynomial $P(t)$ at the endpoints of each sub-interval [G].
4. Detect a sign change in $P(t)$ between adjacent sub-interval endpoints [S1][S2, unofficial].
5. If a sign change is detected within a sub-interval $[t_i, t_{i+1}]$, a root is guaranteed to exist within this range by the Intermediate Value Theorem [G].
6. Set the new search interval to $[t_i, t_{i+1}]$ and repeat until the interval width is below a tolerance $\epsilon$ or a root is isolated [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neubauer_iterative_root_finder(a, b, c, d, t_min, t_max, tolerance=1e-6):
    # [G] Polynomial coefficients and initial interval bounds are assumed provided.
    while t_max - t_min > tolerance:
        # Subdivide current interval into smaller sub-intervals
        # [G] Sources do not specify the exact subdivision count or strategy.
        # A standard approach is binary subdivision or fixed-step refinement.
        t_mid = (t_min + t_max) / 2.0
        p_min = a * t_min**3 + b * t_min**2 + c * t_min + d
        p_mid = a * t_mid**3 + b * t_mid**2 + c * t_mid + d
        if p_min * p_mid < 0:
            t_max = t_mid
        else:
            t_min = t_mid
    return (t_min + t_max) / 2.0
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method yields numerically stable results in single-precision arithmetic, avoiding catastrophic cancellation issues common in analytical cubic solvers [S1][S2, unofficial].
*   **Degenerate Input:** If the ray misses the cell or is tangent, sign change detection may fail; the algorithm assumes a root exists within the initial interval [G].
*   **Multiple Roots:** The method isolates the first sign change, finding the smallest $t$ if subdivision proceeds from $t_{min}$ upwards [G].

## Operating characteristics
*   **Time Complexity:** Logarithmic convergence $O(\log(1/\epsilon))$ with binary subdivision [G].
*   **Memory Complexity:** $O(1)$, requiring only a few floating-point variables [G].
*   **Performance:** Slightly faster than analytical root finders based on Schwarze's cubic solver [S1][S2, unofficial].
*   **Strengths:** Single-precision numerical stability and implementation simplicity [S1][S2, unofficial].
*   **Limitations:** Iterative refinement may be slower than direct analytical solutions when high precision is unnecessary [G].

## Background
Proposed by Marmitt et al. [S1][S2, unofficial], based on the Neubauer iterative root finding technique. It was employed in interactive isosurface ray tracing of large octree volumes, where performance and single-precision stability are critical [S1][S2, unofficial].

## Subtopics

### [de Casteljau-based root finding](1596-de-casteljau-based-root-finding.md)
Uses de Casteljau's algorithm for root isolation and refinement, differing from Neubauer's interval subdivision approach. Choose when Bernstein polynomial evaluation is preferred over direct polynomial evaluation.

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
