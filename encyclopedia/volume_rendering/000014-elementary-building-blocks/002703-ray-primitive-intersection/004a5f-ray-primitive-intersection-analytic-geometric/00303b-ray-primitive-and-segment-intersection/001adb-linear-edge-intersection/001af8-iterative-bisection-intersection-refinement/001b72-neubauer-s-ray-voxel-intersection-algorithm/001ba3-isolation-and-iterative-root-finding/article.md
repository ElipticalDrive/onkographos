# Isolation and Iterative Root Finding

Also called: Isolation method, New intersection algorithm

## Overview
Isolation and Iterative Root Finding is a ray-voxel intersection algorithm for iso-surfaces that combines root isolation via derivative extrema with iterative refinement to achieve both the speed of Neubauer's method and the correctness guarantees of Schwarze's method [S1, unofficial]. It solves the problem of finding the first intersection of a ray with a trivariate polynomial within a voxel by ensuring the search interval contains exactly one root before applying repeated linear interpolation [S1, unofficial]. It sets itself apart from its siblings by explicitly isolating the root interval using the polynomial's extrema, thereby avoiding the false positives that can occur in Neubauer's method when multiple intersections exist within a single segment [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray segment parameter range $[t_{in}, t_{out}]$, and polynomial coefficients $A, B, C, D$ for $f(t) = At^3 + Bt^2 + Ct + D$ [S1, unofficial].
*   **Output:** Intersection parameter $t_{hit}$ or "NO HIT" [S1, unofficial].

**Data Structures:**
*   Scalar variables $t_0, t_1, f_0, f_1, e_0, e_1$ [S1, unofficial].

**Steps:**
1.  **Initialization:** Set $t_0 = t_{in}$, $t_1 = t_{out}$, $f_0 = f(t_0)$, and $f_1 = f(t_1)$ [S1, unofficial].
2.  **Extrema Calculation:** Solve the quadratic derivative equation $f'(t) = 3At^2 + 2Bt + C = 0$ [S1, unofficial]. Calculate the discriminant $\Delta = 4B^2 - 12AC$ [S1, unofficial]. If $\Delta < 0$, there are no real extrema; proceed directly to iterative refinement [S1, unofficial]. If $\Delta \ge 0$, compute the roots $e_0 = \frac{-2B - \sqrt{\Delta}}{6A}$ and $e_1 = \frac{-2B + \sqrt{\Delta}}{6A}$, ensuring $e_0 \le e_1$ [S1, unofficial].
3.  **Root Isolation:** Iterate through the extrema to isolate the first segment containing a root [S1, unofficial]. If $e_0 \in [t_0, t_1]$: compute $f(e_0)$ [S1, unofficial]. If $\text{sign}(f(e_0)) == \text{sign}(f_0)$, advance to the second segment by setting $t_0 := e_0$ and $f_0 := f(e_0)$ [S1, unofficial]. Otherwise, set $t_1 := e_0$ and $f_1 := f(e_0)$ [S1, unofficial]. If $e_1 \in [t_0, t_1]$: compute $f(e_1)$ [S1, unofficial]. If $\text{sign}(f(e_1)) == \text{sign}(f_0)$, advance to the third segment by setting $t_0 := e_1$ and $f_0 := f(e_1)$ [S1, unofficial]. Otherwise, set $t_1 := e_1$ and $f_1 := f(e_1)$ [S1, unofficial].
4.  **No Hit Check:** If $\text{sign}(f_0) == \text{sign}(f_1)$, return "NO HIT" [S1, unofficial].
5.  **Iterative Refinement:** Use repeated linear interpolation to find the root in the isolated interval $[t_0, t_1]$ [S1, unofficial]. Loop $i$ from 1 to $N$ (typically $N=2$ or $3$): compute candidate $t = t_0 + (t_1 - t_0) \frac{-f_0}{f_1 - f_0}$ [S1, unofficial]. If $\text{sign}(f(t)) == \text{sign}(f_0)$, update $t_0 := t$ and $f_0 := f(t)$ [S1, unofficial]. Otherwise, update $t_1 := t$ and $f_1 := f(t)$ [S1, unofficial].
6.  **Return:** Return $t_{hit} = t_0 + (t_1 - t_0) \frac{-f_0}{f_1 - f_0}$ [S1, unofficial].

**Parameters:**
*   $N$: Number of iterations. Typical values are 2 or 3. A fixed number of iterations is preferred over an adaptive termination criterion for performance [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Degenerate Quadratic:** If $A=0$, the polynomial is quadratic and the derivative is linear. The algorithm must handle $3A=0$ separately to avoid division by zero in the quadratic formula [G].
*   **Numerical Stability:** The method provides better numerical stability than Neubauer's method [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sign(x):
    return math.copysign(1, x)

def isolation_and_iterative_root_finding(t_in, t_out, A, B, C, D, N):
    # Step 1: Initialization
    t0 = t_in
    t1 = t_out
    f0 = A * t0**3 + B * t0**2 + C * t0 + D
    f1 = A * t1**3 + B * t1**2 + C * t1 + D

    # Step 2: Find extrema
    if A == 0:
        # Linear derivative: 2Bt + C = 0
        if B != 0:
            e0 = -C / (2 * B)
            e1 = float('inf')
        else:
            e0 = float('inf')
            e1 = float('inf')
    else:
        delta = 4 * B**2 - 12 * A * C
        if delta < 0:
            e0 = float('inf')
            e1 = float('inf')
        else:
            sqrt_delta = math.sqrt(delta)
            e0 = (-2 * B - sqrt_delta) / (6 * A)
            e1 = (-2 * B + sqrt_delta) / (6 * A)

    # Step 3: Root Isolation
    if math.isfinite(e0) and t0 <= e0 <= t1:
        f_e0 = A * e0**3 + B * e0**2 + C * e0 + D
        if sign(f_e0) == sign(f0):
            t0 = e0
            f0 = f_e0
        else:
            t1 = e0
            f1 = f_e0

    if math.isfinite(e1) and t0 <= e1 <= t1:
        f_e1 = A * e1**3 + B * e1**2 + C * e1 + D
        if sign(f_e1) == sign(f0):
            t0 = e1
            f0 = f_e1
        else:
            t1 = e1
            f1 = f_e1

    # Step 4: No Hit Check
    if sign(f0) == sign(f1):
        return None

    # Step 5: Iterative Refinement
    for i in range(1, N + 1):
        if f1 == f0:
            break
        t = t0 + (t1 - t0) * (-f0) / (f1 - f0)
        f_t = A * t**3 + B * t**2 + C * t + D
        if sign(f_t) == sign(f0):
            t0 = t
            f0 = f_t
        else:
            t1 = t
            f1 = f_t

    # Step 6: Return
    if f1 == f0:
        return t0
    return t0 + (t1 - t0) * (-f0) / (f1 - f0)
```

## Subtopics

### [de Casteljau-based root finding](1596-de-casteljau-based-root-finding.md)
A root finding algorithm for polynomials in Bernstein form that isolates and refines the first intersection of a viewing ray with an isosurface by subdividing the control polygon using the de Casteljau algorithm at its intersection with the zero axis. It sets itself apart from knot insertion methods like [MR07] by offering quadratic convergence speed and constant memory usage without index operations, making it significantly faster for GPU architectures. See [de Casteljau-based root finding](1596-de-casteljau-based-root-finding.md).

### [Neubauer Iterative Root Finder](2422-neubauer-iterative-root-finder.md)
The Neubauer Iterative Root Finder is a numerical method for locating the first positive root of the ray-volume intersection polynomial within a voxel cell. It solves the problem of determining the precise entry point of a ray into an implicit surface defined by a scalar field, specifically within the context of octree-based volume rendering. It is distinguished from analytical solvers (such as Schwarze’s cubic solver) by its reliance on iterative bisection and re-parameterization, offering slightly higher performance and robust single-precision numerical stability. See [Neubauer Iterative Root Finder](2422-neubauer-iterative-root-finder.md).

### Sturm Series root finding
A root finding approach based on Sturm Series working in power form [S2]. It uses the Sturm Series to count multiplicities of roots in a given interval and employs binary subdivision of the search interval [S2]. The Sturm Series is computed once at the beginning and evaluated once per iteration [S2]. It offers exponential convergence with constant memory consumption during iteration, but high overall memory consumption as the whole series must be kept in memory [S2]. Numerical problems inherent to power form implementations become more severe with more iterations [S2]. For degree 9 (tri-cubic), it has a relative execution time of 0.077, MSE of 0.003, and max error of 1.0 [S2].

## Operating characteristics
*   **Time Complexity:** Roughly a factor of three faster than Schwarze's method [S1, unofficial].
*   **Memory Complexity:** Minimal scalar usage, making it well suited for a data parallel SIMD implementation [S1, unofficial].
*   **Performance:** Provides the same guarantees on correctness as Schwarze's method while offering better numerical stability [S1, unofficial].
*   **Strengths:** Correctness guarantees via root isolation, high speed, and SIMD friendliness [S1, unofficial].
*   **Limitations:** Requires upfront computation of polynomial coefficients and extrema [S1, unofficial].
*   **When to use:** When both speed and correctness are required for trivariate polynomials within a voxel [S1, unofficial].

## Background
The method was developed to combine the speed of Neubauer's method with the correctness of Schwarze's method by isolating roots via extrema before applying iterative refinement [S1, unofficial]. It leverages the observation that repeated linear interpolation finds the correct root fast and reliably when the start interval contains exactly one root [S1, unofficial].

## Sources
[S1] Fast and Accurate Ray-Voxel Intersection Techniques for Iso-Surface Ray ..., https://www.sci.utah.edu/~wald/Publications/2004/iso/IsoIsec_VMV2004.pdf, fetched 2026-10-29 (unofficial)
[S2] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
