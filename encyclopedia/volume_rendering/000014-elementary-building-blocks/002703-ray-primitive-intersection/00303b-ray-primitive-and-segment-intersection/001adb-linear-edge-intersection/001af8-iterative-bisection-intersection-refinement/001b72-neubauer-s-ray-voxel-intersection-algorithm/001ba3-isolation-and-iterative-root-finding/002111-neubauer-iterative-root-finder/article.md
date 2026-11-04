# Neubauer Iterative Root Finder

Also called: Neubauer's method

## Overview
The Neubauer Iterative Root Finder is a numerical method for locating the first positive root of the ray-volume intersection polynomial within a voxel cell. It solves the problem of determining the precise entry point of a ray into an implicit surface defined by a scalar field, specifically within the context of octree-based volume rendering. It is distinguished from analytical solvers (such as Schwarze’s cubic solver) by its reliance on iterative bisection and re-parameterization, offering slightly higher performance and robust single-precision numerical stability [S1][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a voxel cell (or octree leaf) defined by its spatial bounds. The scalar field values at the cell corners are used to construct the cubic polynomial coefficients $a, b, c, d$ representing the ray's intersection with the cell's boundary or interior function.
*   **Output:** The parameter $t$ along the ray corresponding to the first positive root (intersection point).

**Data Structures:**
The method operates on the cubic polynomial $P(t) = at^3 + bt^2 + ct + d$. No complex data structures are required beyond standard floating-point variables for the ray parameters and polynomial coefficients.

**Algorithm Steps:**
1.  **Initial Interval:** Define the initial search interval for the ray parameter $t$, typically $[t_{min}, t_{max}]$, corresponding to the entry and exit points of the ray with respect to the voxel cell boundaries.
2.  **Iterative Re-parameterization:**
    *   While the interval width is greater than a specified tolerance $\epsilon$:
    *   Subdivide the current interval $[t_{start}, t_{end}]$ into smaller sub-intervals.
    *   Evaluate the polynomial $P(t)$ at the endpoints of these sub-intervals.
    *   Check for a sign change in $P(t)$ between adjacent sub-interval endpoints.
3.  **Root Isolation:**
    *   If a sign change is detected within a sub-interval $[t_i, t_{i+1}]$, a root is guaranteed to exist within this range (by the Intermediate Value Theorem).
    *   Set the new search interval to $[t_i, t_{i+1}]$.
4.  **Refinement:**
    *   Repeat the subdivision and sign-check process on the new, smaller interval.
    *   The process terminates when the interval width is sufficiently small or a sign change is definitively isolated.
5.  **Result:** The final $t$ value is taken as the midpoint of the final isolated interval or the endpoint where the sign change was detected, depending on the specific implementation's convergence criteria.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neubauer_iterative_root_finder(ray_origin, ray_direction, cell_bounds, tolerance=1e-6):
    """
    Finds the first positive root of the ray-cell intersection polynomial
    using iterative re-parameterization and sign-change detection.
    
    Args:
        ray_origin: (x, y, z) origin of the ray.
        ray_direction: (dx, dy, dz) direction of the ray.
        cell_bounds: Object defining the voxel cell geometry.
        tolerance: Convergence threshold for interval width.
        
    Returns:
        t: The parameter value of the intersection.
    """
    # 1. Construct polynomial coefficients a, b, c, d for P(t) = at^3 + bt^2 + ct + d
    # This step depends on the specific scalar field definition and cell geometry.
    # [G] Standard ray-triangle or ray-box intersection polynomial derivation is assumed here.
    a, b, c, d = compute_polynomial_coefficients(ray_origin, ray_direction, cell_bounds)
    
    # 2. Define initial search interval [t_min, t_max]
    # [G] t_min is typically 0 or the entry point into the cell; t_max is the exit point.
    t_min = 0.0
    t_max = compute_exit_parameter(ray_origin, ray_direction, cell_bounds)
    
    if t_max < t_min:
        return None  # No intersection within valid range
        
    # 3. Iterative refinement
    while (t_max - t_min) > tolerance:
        # Subdivide the current interval into smaller sub-intervals
        # [G] The sources do not specify the exact number of subdivisions per step.
        # A common approach is binary subdivision (bisection) or fixed small steps.
        # We assume binary subdivision for robustness as implied by "iterative re-parameterization".
        
        t_mid = (t_min + t_max) / 2.0
        
        # Evaluate polynomial at endpoints of sub-intervals
        p_min = evaluate_polynomial(a, b, c, d, t_min)
        p_mid = evaluate_polynomial(a, b, c, d, t_mid)
        
        # Check for sign change
        if p_min * p_mid < 0:
            # Root is in [t_min, t_mid]
            t_max = t_mid
        else:
            # Root is in [t_mid, t_max] (or no root in this branch, but we assume one exists)
            # [G] If no sign change is found in the left half, the root must be in the right half
            # if we know a root exists in the original interval.
            t_min = t_mid
            
    # 4. Return the estimated root
    return (t_min + t_max) / 2.0

def evaluate_polynomial(a, b, c, d, t):
    return a * t**3 + b * t**2 + c * t + d
```

**Edge Cases and Pitfalls:**
*   **Precision:** The method is noted for yielding numerically stable results in single-precision arithmetic, avoiding the catastrophic cancellation issues that can plague analytical cubic solvers [S1][S2, unofficial].
*   **Degenerate Input:** If the ray is tangent to the surface or misses the cell entirely, the sign change detection may fail. The algorithm assumes a root exists within the initial interval.
*   **Multiple Roots:** The method finds the first sign change. If multiple roots exist within the cell, it finds the smallest $t$ (closest to the ray origin) provided the subdivision strategy explores from $t_{min}$ upwards.

## Operating characteristics
*   **Time Complexity:** The convergence rate depends on the subdivision strategy. With binary subdivision, it is logarithmic with respect to the desired precision ($O(\log(1/\epsilon))$).
*   **Memory Complexity:** $O(1)$, as it only requires storing a few floating-point variables.
*   **Performance:** It is slightly faster than analytical root finders based on Schwarze's cubic solver [S1][S2, unofficial].
*   **Strengths:** Numerical stability in single-precision; simplicity of implementation; no need for complex trigonometric or square-root operations required by analytical solutions.
*   **Limitations:** Iterative nature may be slower than a direct analytical solution if high precision is not required; requires careful handling of the initial interval and sign change detection.

## Background
The Neubauer Iterative Root Finder was proposed by Marmitt et al. [S1][S2, unofficial] as an improvement over analytical cubic solvers for ray-tracing implicit surfaces in large octree volumes. It is based on the Neubauer iterative root finding technique. The method was employed in the context of interactive isosurface ray tracing, where performance and stability in single-precision arithmetic are critical [S1][S2, unofficial].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S2] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
