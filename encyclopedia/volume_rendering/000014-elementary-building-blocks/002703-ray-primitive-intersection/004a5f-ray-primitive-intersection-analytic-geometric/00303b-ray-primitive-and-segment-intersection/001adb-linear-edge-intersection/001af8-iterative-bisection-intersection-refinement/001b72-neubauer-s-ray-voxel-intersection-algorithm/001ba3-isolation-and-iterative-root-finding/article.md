# Isolation and Iterative Root Finding

Also called: Isolation method, New intersection algorithm

## Overview
Isolation and Iterative Root Finding is a ray-voxel intersection algorithm designed to find the first intersection of a viewing ray with an implicit surface (defined by a trivariate polynomial) within a voxel cell. It solves the problem of balancing the high correctness guarantees of analytical solvers with the computational speed of iterative refinement methods. Unlike [Schwarze's Analytic Inversion](1001-schwarze-s-analytic-inversion.md), which computes all roots and is slow, and [Neubauer's Ray-Voxel Intersection Algorithm](975-neubauer-s-ray-voxel-intersection-algorithm.md), which may fail to locate valid intersections or return incorrect ones in complex cases, this method isolates the root interval using polynomial extrema before applying iterative refinement. This ensures that the final interval contains exactly one root, providing the correctness of Schwarze's method with a speed roughly three times faster [S1, unofficial].

## How it works
The method operates on a ray segment $[t_{in}, t_{out}]$ parameterized by $t$, where the implicit function is $f(t)$. The algorithm proceeds in two phases: root isolation via extrema analysis, followed by iterative root refinement.

### Inputs and Outputs
- **Inputs**: Ray segment endpoints $t_{in}, t_{out}$; polynomial coefficients $A, B, C, D$ for $f(t) = At^3 + Bt^2 + Ct + D$; isovalue $\rho_{iso}$ (or offset $f_{iso} = \rho_{iso}$ if $f$ represents density).
- **Outputs**: Intersection parameter $t_{hit}$ or "NO HIT".

### Data Structures
No complex data structures are required. The algorithm uses scalar variables for parameters and coefficients.

### Steps
1. **Initialization**:
   Set $t_0 = t_{in}$, $t_1 = t_{out}$.
   Compute $f_0 = f(t_0)$ and $f_1 = f(t_1)$.
   *(Note: $f(t)$ is the polynomial value minus the isovalue, i.e., $f(t) = P(t) - \rho_{iso}$)*.

2. **Extrema Isolation**:
   Compute the derivative $f'(t) = 3At^2 + 2Bt + C$.
   Solve the quadratic equation $f'(t) = 0$ for real roots.
   Let the roots be $e_0$ (smaller) and $e_1$ (larger). If no real roots exist, skip to Step 4.

3. **Segment Advancement**:
   Iterate through the extrema to isolate the interval containing the first root.
   
   - **First Extremum ($e_0$)**:
     If $e_0 \in [t_0, t_1]$:
     - Compute $f(e_0)$.
     - If $\text{sign}(f(e_0)) == \text{sign}(f_0)$:
       The root is in the segment after $e_0$. Advance:
       $t_0 := e_0$
       $f_0 := f(e_0)$
     - Else:
       The root is in the segment before $e_0$. Constrain:
       $t_1 := e_0$
       $f_1 := f(e_0)$
   
   - **Second Extremum ($e_1$)**:
     If $e_1 \in [t_0, t_1]$:
     - Compute $f(e_1)$.
     - If $\text{sign}(f(e_1)) == \text{sign}(f_0)$:
       The root is in the segment after $e_1$. Advance:
       $t_0 := e_1$
       $f_0 := f(e_1)$
     - Else:
       The root is in the segment before $e_1$. Constrain:
       $t_1 := e_1$
       $f_1 := f(e_1)$

4. **Hit Test**:
   If $\text{sign}(f_0) == \text{sign}(f_1)$, return **NO HIT**.
   Otherwise, the interval $[t_0, t_1]$ contains exactly one root.

5. **Iterative Refinement**:
   Apply repeated linear interpolation (regula falsi) for a fixed number of iterations $N$ (typically 2–3).
   
   For $i = 1$ to $N$:
   - Compute candidate $t$:
     $$ t = t_0 + (t_1 - t_0) \frac{-f_0}{f_1 - f_0} $$
   - Compute $f(t)$.
   - If $\text{sign}(f(t)) == \text{sign}(f_0)$:
     $t_0 := t$
     $f_0 := f(t)$
   - Else:
     $t_1 := t$
     $f_1 := f(t)$

6. **Return Result**:
   $$ t_{hit} = t_0 + (t_1 - t_0) \frac{-f_0}{f_1 - f_0} $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def isolation_and_iterative_root_finding(t_in, t_out, A, B, C, D, rho_iso):
    # f(t) = A*t^3 + B*t^2 + C*t + D - rho_iso
    # f'(t) = 3*A*t^2 + 2*B*t + C
    
    t0 = t_in
    t1 = t_out
    f0 = A*t0**3 + B*t0**2 + C*t0 + D - rho_iso
    f1 = A*t1**3 + B*t1**2 + C*t1 + D - rho_iso
    
    # Solve 3At^2 + 2Bt + C = 0
    a_quad = 3 * A
    b_quad = 2 * B
    c_quad = C
    
    discriminant = b_quad**2 - 4 * a_quad * c_quad
    
    if discriminant < 0:
        # No real extrema, proceed directly to refinement if signs differ
        pass
    else:
        sqrt_disc = sqrt(discriminant)
        e0 = (-b_quad - sqrt_disc) / (2 * a_quad)
        e1 = (-b_quad + sqrt_disc) / (2 * a_quad)
        
        # Process first extremum e0
        if t0 <= e0 <= t1:
            f_e0 = A*e0**3 + B*e0**2 + C*e0 + D - rho_iso
            if sign(f_e0) == sign(f0):
                t0 = e0
                f0 = f_e0
            else:
                t1 = e0
                f1 = f_e0
        
        # Process second extremum e1
        if t0 <= e1 <= t1:
            f_e1 = A*e1**3 + B*e1**2 + C*e1 + D - rho_iso
            if sign(f_e1) == sign(f0):
                t0 = e1
                f0 = f_e1
            else:
                t1 = e1
                f1 = f_e1

    if sign(f0) == sign(f1):
        return NO_HIT

    # Iterative refinement (Regula Falsi)
    N = 3 # Typical value
    for i in range(1, N + 1):
        # Avoid division by zero if f1 == f0, though unlikely in valid root interval
        if f1 == f0:
            t = (t0 + t1) / 2.0
        else:
            t = t0 + (t1 - t0) * (-f0) / (f1 - f0)
        
        f_t = A*t**3 + B*t**2 + C*t + D - rho_iso
        
        if sign(f_t) == sign(f0):
            t0 = t
            f0 = f_t
        else:
            t1 = t
            f1 = f_t

    # Final estimate
    if f1 == f0:
        t_hit = (t0 + t1) / 2.0
    else:
        t_hit = t0 + (t1 - t0) * (-f0) / (f1 - f0)
        
    return t_hit

def sign(x):
    if x > 0: return 1
    elif x < 0: return -1
    else: return 0
```

### Parameters
- **N (Iterations)**: Typically 2–3. A fixed number is faster than an adaptive termination criterion [S1, unofficial].

### Edge Cases and Pitfalls
- **No Real Extrema**: If the discriminant of the derivative is negative, the function is monotonic. The algorithm skips the isolation steps and proceeds to refinement.
- **Extrema Outside Interval**: Extrema outside $[t_0, t_1]$ are ignored.
- **Sign Equality**: If $f_0$ and $f_1$ have the same sign after isolation, no root exists in the segment.
- **Division by Zero**: If $f_1 \approx f_0$ during refinement, the denominator becomes small. The pseudocode handles this by falling back to bisection (midpoint) [G].
- **Multiple Roots**: The method isolates the *first* root. If multiple roots exist in the voxel, the extrema analysis ensures the interval $[t_0, t_1]$ contains only one, guaranteeing the correct first intersection [S1, unofficial].

## Subtopics

### [de Casteljau-based root finding](1596-de-casteljau-based-root-finding.md)
This method finds the intersection of a viewing ray with an isosurface defined by a trivariate polynomial in Bernstein form by iteratively subdividing the control polygon using the de Casteljau algorithm at the intersection with the zero axis. It distinguishes itself from sibling methods like [Neubauer Iterative Root Finder](2422-neubauer-iterative-root-finder.md) and [Knot insertion root finding](#knot-insertion-root-finding) by offering quadratic convergence speed and constant memory usage without index operations, making it significantly faster and more suitable for GPU architectures than linear-convergence knot insertion approaches.

### [Neubauer Iterative Root Finder](2422-neubauer-iterative-root-finder.md)
The Neubauer Iterative Root Finder is a numerical method for finding the intersection of a ray with an implicit surface (typically defined by a trivariate polynomial or spline) within a voxel cell. It solves the problem of determining the parameter $t$ along the ray where the implicit function $f(t)$ equals zero. It sets itself apart from analytical solvers like Schwarze’s cubic solver by using iterative bisection/refinement, which yields single-precision, numerically stable results and is slightly faster in practice. It builds on [Isolation and Iterative Root Finding](1041-isolation-and-iterative-root-finding.md) by using root isolation via sign changes within sub-intervals.

### Sturm Series root finding
A root finding approach based on Sturm Series working in power form [S2]. It uses the Sturm Series to count multiplicities of roots in a given interval and employs binary subdivision of the search interval [S2]. The algorithm computes the Sturm Series at the beginning, evaluates it once per iteration, and subdivides the interval. It offers exponential convergence with constant memory consumption during iteration, but high overall memory consumption as the whole Sturm Series must be kept in memory [S2]. Numerical problems inherent to power form implementations become more severe with more iterations [S2]. For degree 9 (tri-cubic), it has a relative execution time of 0.077, MSE of 0.003, and max error of 1.0 [S2].

## Operating characteristics
- **Time Complexity**: Roughly a factor of three faster than Schwarze's analytical method [S1, unofficial]. The cost is dominated by solving the quadratic derivative and evaluating the cubic polynomial.
- **Memory Complexity**: Constant memory usage, requiring only scalar storage for coefficients and parameters.
- **Performance**: On degree 9 (tri-cubic) polynomials, it provides high accuracy with low relative execution time compared to other methods [S2].
- **Strengths**: Guarantees correctness (one root in the final interval), better numerical stability than Neubauer's method, and SIMD-friendly control flow due to simple branching [S1, unofficial].
- **Limitations**: Requires computing polynomial coefficients and extrema upfront. Not suitable if the polynomial form is not available or if the cost of coefficient computation outweighs the intersection benefit.
- **When to Use**: When high performance and correctness are both required, especially in SIMD/GPU environments.
- **When Not to Use**: When the polynomial coefficients are not pre-computed or when the overhead of computing them is prohibitive.

## Background
The method was introduced by Wald et al. in 2004 as a "new intersection algorithm" to address the trade-off between speed and correctness in ray-voxel intersection for isosurface rendering [S1, unofficial]. It builds on the observation that repeated linear interpolation is reliable if the starting interval contains exactly one root. By isolating roots via extrema, it guarantees this condition. The method is well-suited for data parallel SIMD implementations, as it avoids the complex control flow of analytical solvers like Schwarze's [S1, unofficial]. It is part of a broader effort to optimize volume rendering for large voxel datasets [S2].

## Sources
[S1] Fast and Accurate Ray-Voxel Intersection Techniques for Iso-Surface Ray ..., https://www.sci.utah.edu/~wald/Publications/2004/iso/IsoIsec_VMV2004.pdf, fetched 2026-10-29 (unofficial)
[S2] Wavelet-based Multiresolution Isosurface Rendering, https://doi.org/10.2312/vg/vg10/013-020, fetched 2026-10-26
