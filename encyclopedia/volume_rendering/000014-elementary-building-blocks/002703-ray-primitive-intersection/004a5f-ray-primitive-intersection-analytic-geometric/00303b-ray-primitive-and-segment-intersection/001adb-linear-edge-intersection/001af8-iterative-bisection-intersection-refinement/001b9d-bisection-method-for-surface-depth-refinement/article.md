# Bisection Method for Surface Depth Refinement

Also called: Bisection Method, Iterative Bisection, Midpoint (Bisection) Algorithm

## Overview
The Bisection Method for Surface Depth Refinement is an iterative root-finding algorithm that determines the exact intersection point of a ray with an implicit surface within a segment known to contain an intersection [S2, unofficial]. It serves as an alternative to the Secant method in ray marching pipelines [S1, unofficial]. The method systematically halves the search interval at each step until the midpoint evaluates to zero [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Two endpoints of a line segment, $P_{low}$ and $P_{high}$, bracketing the intersection [S2, unofficial].
*   **Output:** A point $P_{hit}$ on the segment where the field function evaluates to zero [S2, unofficial].

**Data Structures:**
*   **Endpoints:** 3D vectors or 1D coordinates along an axis-aligned edge [S2, unofficial].
*   **Field Values:** Scalars $f(P_{low})$ and $f(P_{high})$ indicating signed distance or occupancy relative to the isosurface threshold [S2, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Set $P_{low}$ and $P_{high}$ to the segment endpoints and evaluate $f_{low} = f(P_{low})$ and $f_{high} = f(P_{high})$ [S2, unofficial].
2.  **Midpoint Calculation:** Compute $P_{mid} = (P_{low} + P_{high}) / 2$ [S2, unofficial].
3.  **Evaluation:** Evaluate the field function at the midpoint: $f_{mid} = f(P_{mid})$ [S2, unofficial].
4.  **Convergence Check:** If $f_{mid} = 0$, the exact intersection point is found [S2, unofficial].
5.  **Interval Update:** Replace the endpoint whose field value has the same sign as $f_{mid}$ with the midpoint [S2, unofficial].
6.  **Iteration:** Repeat from Step 2 until convergence [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bisection_intersection(P_low, P_high, field_function):
    """Refines the intersection point on a segment using the bisection method."""
    f_low = field_function(P_low)
    f_high = field_function(P_high)
    while True:
        P_mid = (P_low + P_high) / 2.0
        f_mid = field_function(P_mid)
        if f_mid == 0:
            return P_mid
        if f_mid * f_low > 0:
            P_low = P_mid
            f_low = f_mid
        else:
            P_high = P_mid
            f_high = f_mid
        # Convergence check (tolerance/max_iter) not specified in sources [G]
```

**Parameters:**
*   **Tolerance:** Not given in the sources [S2, unofficial].
*   **Max Iterations:** Not given in the sources [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Bracketing Segments:** The method requires the initial segment to bracket a root [S2, unofficial].
*   **Discontinuous Functions:** The method requires continuity [S2, unofficial].
*   **Multiple Roots:** Not given in the sources [S2, unofficial].

## Subtopics

### [Bracketed Newton Bisection](3349-bracketed-newton-bisection.md)
Combines bisection with Newton’s method for high-degree polynomials, ensuring all real roots are found by isolating them in monotonic intervals. Choose when dealing with complex implicit surfaces defined by high-degree polynomials where simple bisection might miss roots or converge slowly.

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S2, unofficial].
*   **Memory Complexity:** Not given in the sources [S2, unofficial].
*   **Performance:** The false position algorithm is "no worse than the midpoint algorithm, and usually better" in terms of iteration count [S2, unofficial].
*   **Strengths:** Guaranteed convergence for continuous functions by systematically halving the search interval [S1, S2, unofficial].
*   **Limitations:** Requires the field function to be continuous and the initial segment to bracket a root [S2, unofficial].
*   **When to Use:** Not given in the sources [S2, unofficial].
*   **When Not to Use:** Not given in the sources [S2, unofficial].

## Background
The bisection method is a classic root-finding algorithm [S1, unofficial]. In isosurface extraction, it is cited as a "far better choice" than simple linear interpolation for finding exact intersection points on edges [S2, unofficial]. It is implemented in the `IsoMesher` class methods `intersect_xaxis()`, `intersect_yaxis()`, and `intersect_zaxis()`, controlled by the preprocessor symbol `BISECTION` [S2, unofficial]. It is also used in differentiable volumetric rendering pipelines as an alternative to the Secant method [S1, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, http://www.oocities.org/tzukkers/isosurf/isosurfaces.html, fetched 2026-10-30 (unofficial)
