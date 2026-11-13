# Neubauer Iterative Root Finder

Also called: Neubauer's method

## Overview
The Neubauer Iterative Root Finder is a numerical method for finding the intersection of a ray with an implicit surface (typically defined by a trivariate polynomial or spline) within a voxel cell. It solves the problem of determining the parameter $t$ along the ray where the implicit function $f(t)$ equals zero. It sets itself apart from analytical solvers like Schwarze’s cubic solver by using iterative bisection/refinement, which yields single-precision, numerically stable results and is slightly faster in practice [S1][S2, unofficial]. It builds on [Isolation and Iterative Root Finding](1041-isolation-and-iterative-root-finding.md) by using root isolation via sign changes within sub-intervals.

## How it works
**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Implicit surface function $f(t)$, where $t$ is the ray parameter.
- Initial search interval $[t_{min}, t_{max}]$ (typically the entry and exit parameters of the current voxel/cell).
- Convergence tolerance $\epsilon$ (e.g., $4.0 \cdot 10^{-3}$) [S3, unofficial].
- Maximum number of iterations $N_{max}$ (e.g., 50) [S3, unofficial].

**Outputs:**
- The smallest positive root $t^* \in [t_{min}, t_{max}]$ such that $f(t^*) \approx 0$.

**Data Structures:**
- No complex data structures required. Standard floating-point variables for interval bounds and function evaluations.

**Steps:**
1. **Evaluate Endpoints:** Compute $f_{min} = f(t_{min})$ and $f_{max} = f(t_{max})$.
2. **Check for Root:** If $f_{min}$ and $f_{max}$ have the same sign (i.e., $f_{min} \cdot f_{max} > 0$), no root is guaranteed in this interval. Return failure or handle as no-intersection [S1][S2, unofficial].
3. **Iterative Refinement:**
   - Initialize $t_{left} = t_{min}$, $t_{right} = t_{max}$.
   - Initialize $f_{left} = f_{min}$, $f_{right} = f_{max}$.
   - Loop for $i = 0$ to $N_{max}$:
     - Calculate midpoint: $t_{mid} = \frac{t_{left} + t_{right}}{2}$.
     - Evaluate function: $f_{mid} = f(t_{mid})$.
     - Check convergence: If $|t_{right} - t_{left}| < \epsilon$, return $t_{mid}$ as the root [S3, unofficial].
     - Determine sub-interval:
       - If $f_{left} \cdot f_{mid} < 0$: The root lies in $[t_{left}, t_{mid}]$. Set $t_{right} = t_{mid}$, $f_{right} = f_{mid}$.
       - Else if $f_{mid} \cdot f_{right} < 0$: The root lies in $[t_{mid}, t_{right}]$. Set $t_{left} = t_{mid}$, $f_{left} = f_{mid}$.
       - Else: If $f_{mid} \approx 0$, return $t_{mid}$. If signs are not opposite, the method may fail to isolate the root in this step (degenerate case) [S1][S2, unofficial].
4. **Return Result:** If the loop completes without convergence, return the last $t_{mid}$ or indicate failure.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neubauer_root_finder(f, t_min, t_max, epsilon=4.0e-3, max_iter=50):
    """
    Finds the root of f(t) in [t_min, t_max] using iterative bisection.
    f: callable, returns scalar value for parameter t.
    """
    f_left = f(t_min)
    f_right = f(t_max)
    
    # Check if a root exists in the interval
    if f_left * f_right > 0:
        return None  # No sign change, root not guaranteed
    
    # Handle exact zero at boundaries
    if abs(f_left) < epsilon:
        return t_min
    if abs(f_right) < epsilon:
        return t_max

    for _ in range(max_iter):
        t_mid = (t_left + t_right) / 2.0
        f_mid = f(t_mid)
        
        # Check convergence based on interval width
        if (t_right - t_left) < epsilon:
            return t_mid
        
        # Check if midpoint is root
        if abs(f_mid) < epsilon:
            return t_mid
            
        # Bisection step
        if f_left * f_mid < 0:
            t_right = t_mid
            f_right = f_mid
        else:
            t_left = t_mid
            f_left = f_mid
            
    return (t_left + t_right) / 2.0  # Return best estimate if max iter reached
```

**Parameters:**
- $\epsilon$: Convergence threshold. Typical value: $4.0 \cdot 10^{-3}$ [S3, unofficial].
- $N_{max}$: Maximum iterations. Typical value: 50 [S3, unofficial].

**Edge Cases and Pitfalls:**
- **No Sign Change:** If the ray grazes the surface or misses it, $f(t_{min})$ and $f(t_{max})$ may have the same sign. The method must detect this and return failure [S1][S2, unofficial].
- **Multiple Roots:** The basic bisection finds one root. If multiple intersections exist in the interval, it finds the first one encountered by the bisection logic.
- **Precision:** Single-precision arithmetic is supported and stable, unlike some analytical cubic solvers which may suffer from catastrophic cancellation [S1][S2, unofficial].

## Subtopics

### Newton–Raphson Voxel Intersection
A variant that replaces the repeated linear interpolation (bisection) of Neubauer's method with Newton–Raphson iteration, while still using the polynomial split from Marmitt et al. [2004] [S3, unofficial]. It uses the same maximum iterations (50) and epsilon ($\epsilon = 4.0 \cdot 10^{-3}$) [S3, unofficial]. It is generally faster but requires the derivative of the implicit function, which may not always be available or cheap to compute [S3, unofficial].

### Boundary Intersection Handling
A technique for handling cases where the ray intersects the boundary of a volume block, which can cause surface approximation to miss intersections in tessellation [S4]. It also handles spline degeneracies where the Jacobian $J_\phi$ is singular at the entry point [S4]. For Explicit Runge-Kutta methods, solution values are clamped to the parameter domain $P$ [S4]. For Newton-Raphson based root finding, if the Jacobian is singular, a new entry point $\tilde{g}_{In}$ is found by shrinking the block: $\tilde{g}_{In} = g_{In} + \delta \frac{p_{Out} - p_{In}}{\|p_{Out} - p_{In}\|_{L2}}$ with a suitable $\delta > 0$ [S4]. The new entry point $\tilde{p}_{In}$ in parameter domain $P$ is found using Newton-Raphson with a shifted starting point: $x_0 = p_{In} + \epsilon \frac{p_{Out} - p_{In}}{\|p_{Out} - p_{In}\|_{L2}}$ [S4]. If Newton-Raphson fails to converge and repeatedly tries to exit the parameter domain, the algorithm detects this and steps along the boundary of the parameter domain until the view-ray is within the domain again [S4].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of iterations (typically 50) [S3, unofficial]. Each iteration involves one function evaluation and basic arithmetic.
- **Memory Complexity:** $O(1)$, requiring only a few scalar variables [S1][S2, unofficial].
- **Performance:** Slightly faster than analytical Schwarze cubic solver [S1][S2, unofficial].
- **Strengths:** Numerically stable in single-precision; simple to implement; no complex algebraic formulas [S1][S2, unofficial].
- **Limitations:** Requires function evaluations; may fail if no sign change is detected; not suitable for finding all roots in an interval if multiple exist [S1][S2, unofficial].
- **When to Use:** When single-precision stability is critical and the function is easy to evaluate [S1][S2, unofficial].
- **When Not to Use:** When the derivative is available and Newton-Raphson is preferred for faster convergence, or when analytical solutions are required for exactness [S3, unofficial].

## Background
Proposed by Marmitt et al. [13] (also cited as [75] in [S2, unofficial]), based on the Neubauer iterative root finder [S1][S2, unofficial]. It was used in the context of interactive isosurface ray tracing of large octree volumes [S1][S2, unofficial]. The method was compared against analytical solvers like Schwarze's cubic solver [S1][S2, unofficial].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S3] Ray tracing of signed distance function grids, https://www.jcgt.org/published/0011/03/06/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
[S4] Interactive Isogeometric Volume Visualization with Pixel-Accurate Geometry, https://doi.org/10.1109/tvcg.2015.2430337, fetched 2026-10-28
