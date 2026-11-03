# Isolation and Iterative Root Finding

Also called: Isolation method, New intersection algorithm

## Overview
Isolation and Iterative Root Finding is a ray-voxel intersection algorithm designed to locate the first intersection of a viewing ray with an iso-surface within a voxel [S1]. It solves the problem of balancing computational speed with numerical correctness, addressing the inaccuracies of Neubauer's method and the slowness of Schwarze's method [S1]. It sets itself apart by first isolating the root interval using polynomial extrema analysis, guaranteeing exactly one root in the final interval, and then applying iterative refinement [S1]. This approach is roughly three times faster than Schwarze's method while providing the same correctness guarantees and better numerical stability [S1].

## How it works
The method operates on a cubic polynomial $f(t)$ representing the density along the ray segment $[t_{in}, t_{out}]$ [S1]. The algorithm consists of two phases: root isolation via extrema analysis, followed by iterative root finding [S1].

**Inputs:**
- Ray segment endpoints $t_{in}$ and $t_{out}$ [S1].
- Polynomial coefficients $A, B, C, D$ for $f(t) = At^3 + Bt^2 + Ct + D$ [S1].

**Outputs:**
- Intersection parameter $t_{hit}$ or "NO HIT" [S1].

**Data Structures:**
- Floating-point variables for $t, f, e$ (extrema) [S1].
- Quadratic solver for $f'(t)$ [S1].

**Steps:**

1.  **Initialization:**
    Set $t_0 = t_{in}$, $t_1 = t_{out}$ [S1].
    Evaluate $f_0 = f(t_0)$ and $f_1 = f(t_1)$ [S1].

2.  **Extrema Analysis:**
    Compute the derivative $f'(t) = 3At^2 + 2Bt + C$ [S1].
    Solve the quadratic equation $f'(t) = 0$ for real roots [S1].
    Let the roots be $e_0$ and $e_1$ with $e_0 \le e_1$ [S1]. If no real roots exist, skip to Step 4 [S1].

3.  **Interval Isolation:**
    Iterate through the extrema to narrow the search interval $[t_0, t_1]$ to a segment containing exactly one root [S1].
    
    For each extremum $e \in \{e_0, e_1\}$:
    - If $e$ is not in $[t_0, t_1]$, continue to the next extremum [S1].
    - If $e$ is in $[t_0, t_1]$:
        - Evaluate $f(e)$ [S1].
        - If $\text{sign}(f(e)) == \text{sign}(f_0)$:
            - The extremum is on the same side of the iso-surface as the start of the current segment [S1]. Advance the start of the segment:
              $t_0 := e$
              $f_0 := f(e)$ [S1].
        - Else:
            - The extremum is on the opposite side [S1]. The root lies between the previous start and this extremum [S1]. Set the end of the segment:
              $t_1 := e$
              $f_1 := f(e)$ [S1].
    
    *Note: The algorithm processes $e_0$ then $e_1$. If $e_0$ splits the interval, the subsequent check for $e_1$ operates on the new $[t_0, t_1]$ [S1].*

4.  **Hit Check:**
    If $\text{sign}(f_0) == \text{sign}(f_1)$, return "NO HIT" [S1].
    Otherwise, the interval $[t_0, t_1]$ contains exactly one root [S1].

5.  **Iterative Refinement:**
    Use repeated linear interpolation (regula falsi) to find the root within $[t_0, t_1]$ [S1].
    Loop $i$ from 1 to $N$ (typically $N=2$ or $3$):
    - Compute candidate $t$:
      $$ t = t_0 + (t_1 - t_0) \frac{-f_0}{f_1 - f_0} $$
    - Evaluate $f(t)$ [S1].
    - If $\text{sign}(f(t)) == \text{sign}(f_0)$:
      $t_0 := t$
      $f_0 := f(t)$ [S1].
    - Else:
      $t_1 := t$
      $f_1 := f(t)$ [S1].

6.  **Return Result:**
    Return $t_{hit} = t_0 + (t_1 - t_0) \frac{-f_0}{f_1 - f_0}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def isolation_and_iterative_root_finding(t_in, t_out, A, B, C, D, N=3):
    def f(t):
        return A * t**3 + B * t**2 + C * t + D

    def sign(x):
        return 1 if x >= 0 else -1

    t0 = t_in
    t1 = t_out
    f0 = f(t0)
    f1 = f(t1)

    # Find extrema by solving f'(t) = 3At^2 + 2Bt + C = 0
    a_quad = 3 * A
    b_quad = 2 * B
    c_quad = C
    discriminant = b_quad**2 - 4 * a_quad * c_quad

    if discriminant >= 0:
        sqrt_disc = math.sqrt(discriminant)
        e0 = (-b_quad - sqrt_disc) / (2 * a_quad)
        e1 = (-b_quad + sqrt_disc) / (2 * a_quad)
        extrema = [e0, e1] if e0 <= e1 else [e1, e0]

        for e in extrema:
            if t0 <= e <= t1:
                f_e = f(e)
                if sign(f_e) == sign(f0):
                    t0 = e
                    f0 = f_e
                else:
                    t1 = e
                    f1 = f_e

    if sign(f0) == sign(f1):
        return None  # NO HIT

    # Iterative refinement using repeated linear interpolation
    for _ in range(N):
        denom = f1 - f0
        if abs(denom) < 1e-12:  # Guard against division by zero [G]
            break
        t = t0 + (t1 - t0) * (-f0) / denom
        f_t = f(t)
        if sign(f_t) == sign(f0):
            t0 = t
            f0 = f_t
        else:
            t1 = t
            f1 = f_t

    denom = f1 - f0
    if abs(denom) < 1e-12:
        t_hit = t0
    else:
        t_hit = t0 + (t1 - t0) * (-f0) / denom
    return t_hit
```

**Parameters:**
- Number of iterations $N$: Typically 2 or 3 [S1]. A fixed number is usually faster than an adaptive termination criterion [S1].

**Edge Cases and Pitfalls:**
- **Three Intersections:** Neubauer's method can falsely return the last intersection if three intersections are contained within a voxel and two are in the first segment [S1]. Isolation and Iterative Root Finding avoids this by explicitly isolating the first root interval using extrema, ensuring the returned root is the first one [S1].
- **Precision:** The method relies on evaluating the polynomial [S1]. Since coefficients are known, this avoids costly trilinear interpolation during refinement, improving speed and stability [S1].
- **No Real Extrema:** If the discriminant is negative, the polynomial is monotonic in the interval, and the initial $[t_{in}, t_{out}]$ is used directly [S1].
- **Division by Zero:** If $f_1$ converges to $f_0$ during refinement, the denominator $f_1 - f_0$ becomes zero; the implementation guards against this by checking the denominator magnitude before division [G].

## Subtopics

### [de Casteljau-based root finding](1596-de-casteljau-based-root-finding.md)
A root finding algorithm for polynomials in Bernstein form that isolates and refines the first intersection of a viewing ray with an isosurface by subdividing the control polygon using the de Casteljau algorithm at its intersection with the zero axis. It sets itself apart from knot insertion methods like [MR07] by offering quadratic convergence speed and constant memory usage without index operations, making it significantly faster for GPU architectures.

### Sturm Series root finding
A root finding approach based on Sturm Series working in power form [S2]. A root-finder in power form using the Sturm Series to count multiplicities of roots in a given interval [S2]. It uses binary subdivision of the search interval [S2]. The algorithm computes the Sturm Series at the beginning, evaluates it once during every iteration step, and subdivides the search interval in a binary manner [S2]. Numerical problems natural for power form implementations get more severe for more iterations [S2]. It exhibits exponential convergence with constant memory consumption during iteration, but high memory consumption overall as the whole Sturm Series needs to be kept in memory during the entire execution [S2]. For degree 9 (tri-cubic), execution time is 0.077 (relative), MSE is 0.003, and max err is 1.0 [S2].

## Operating characteristics
- **Time Complexity:** Roughly a factor of three faster than Schwarze code while providing same guarantees on correctness and better numerical stability [S1]. The cost is dominated by solving the quadratic equation for extrema and evaluating the polynomial [S1].
- **Memory Complexity:** Low for the isolation method [S1]. High for Sturm Series variant as the whole series is kept in memory [S2].
- **Measured Performance:** For degree 9 (tri-cubic) polynomials, the method (referred to as "RF" or Regula Falsi in comparisons) shows relative execution time of 0.062-0.077 compared to Schwarze's 1.0, with low Mean Squared Error (0.003-0.016) and max error (0.66-1.0) [S2].
- **Strengths:** Correctness guarantees similar to Schwarze's method; better numerical stability than Neubauer's; well-suited for SIMD parallel implementation due to little branching and regular control flow [S1].
- **Limitations:** Requires computing polynomial coefficients and extrema upfront [S1]. Numerical problems inherent to power form implementations can become more severe with more iterations if using Sturm Series variants [S2].
- **When to use:** When high performance is required but correctness cannot be compromised, such as in real-time volume rendering of smooth iso-surfaces [S1].
- **When not to use:** If the overhead of computing coefficients is prohibitive and lower precision is acceptable [S1].

## Background
The method was derived by Wald et al. in 2004 as a response to the trade-offs between existing ray-voxel intersection techniques [S1]. It aims to combine the speed of Neubauer's method with the correctness of Schwarze's method [S1]. The approach leverages the fact that repeated linear interpolation is fast and reliable if the start interval contains exactly one root, which is guaranteed by the prior extrema analysis [S1]. It is particularly effective for SIMD implementations, as demonstrated by Wald et al., who showed that modern SIMD instruction sets can significantly improve performance for this algorithm due to its data-parallel nature and lack of complex control flow [S1].

## Sources
[S1] Fast and Accurate Ray-Voxel Intersection Techniques for Iso-Surface Ray ..., https://www.sci.utah.edu/~wald/Publications/2004/iso/IsoIsec_VMV2004.pdf, fetched 2026-10-29
[S2] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
