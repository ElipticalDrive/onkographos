# Cubic Hermite Interpolation

Also called: Third Order Polynomial, Cubic polynomial interpolation

## Overview
Cubic Hermite Interpolation is a method for determining precise isovertex positions along zero-crossing edges in contouring algorithms by fitting a cubic polynomial to both function values and gradients (Hermite data) at the edge endpoints [S2]. It improves upon linear interpolation by reducing artifacts caused by the non-linear nature of implicit functions, offering higher accuracy at the cost of solving for polynomial coefficients and finding cubic roots [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two scalar field values $v_0 = f(x_0)$ and $v_1 = f(x_1)$ at edge endpoints $x_0$ and $x_1$; two derivative estimates $d_0 = f'(x_0)$ and $d_1 = f'(x_1)$; and the target isovalue $k$ [S1, S2].
*   **Outputs:** The position $x^*$ along the edge where the interpolated function equals $k$ [S1].

**Data Structures:**
The method operates on scalar values and a single cubic polynomial defined by four coefficients [S2].

**Steps:**
1.  **Compute Coefficients:** Determine the coefficients $a_0, a_1, a_2, a_3$ of the cubic polynomial $p(x) = a_0 + a_1 x + a_2 x^2 + a_3 x^3$ such that it satisfies the Hermite constraints:
    $$
    \begin{aligned}
    p(x_0) &= v_0 \\
    p(x_1) &= v_1 \\
    p'(x_0) &= d_0 \\
    p'(x_1) &= d_1
    \end{aligned}
    $$
    The closed-form solution for these coefficients is [S2]:
    $$
    \begin{aligned}
    a_0 &= v_0 \\
    a_1 &= d_0 \\
    a_2 &= 3v_1 - 3v_0 - 2d_0 - d_1 \\
    a_3 &= 2v_0 - 2v_1 + d_0 + d_1
    \end{aligned}
    $$
    These formulas assume the edge is parameterized such that $x_0=0$ and $x_1=1$ [S2].

2.  **Solve for Roots:** Find all real roots of the equation $p(x) = k$, which is equivalent to finding roots of $q(x) = a_0 + a_1 x + a_2 x^2 + a_3 x^3 - k = 0$ [S1, S2].

3.  **Select Isovertex:** If multiple real roots exist within the edge interval, select the "middle" root to uniquely define the isovertex position [S2]. This choice is based on the observation that a zero-crossing edge must contain an odd number of roots, and the middle root best represents the primary crossing [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cubic_hermite_interpolation(v0, v1, d0, d1, k):
    """
    Computes the isovertex position along an edge [0, 1] using cubic Hermite interpolation.
    
    Args:
        v0: Function value at x=0
        v1: Function value at x=1
        d0: Derivative estimate at x=0
        d1: Derivative estimate at x=1
        k: Target isovalue
        
    Returns:
        x_star: The position of the isovertex along the edge [0, 1]
    """
    # Step 1: Compute cubic coefficients
    # p(x) = a0 + a1*x + a2*x^2 + a3*x^3
    a0 = v0
    a1 = d0
    a2 = 3 * v1 - 3 * v0 - 2 * d0 - d1
    a3 = 2 * v0 - 2 * v1 + d0 + d1
    
    # Adjust for isovalue k: solve a0 + a1*x + a2*x^2 + a3*x^3 = k
    # Equivalent to: a3*x^3 + a2*x^2 + a1*x + (a0 - k) = 0
    c3 = a3
    c2 = a2
    c1 = a1
    c0 = a0 - k
    
    # Step 2: Find real roots of the cubic equation
    roots = solve_cubic_real(c0, c1, c2, c3)
    
    # Step 3: Filter roots within the edge interval [0, 1]
    valid_roots = [r for r in roots if 0.0 <= r <= 1.0]
    
    if not valid_roots:
        # Fallback: If no roots found, return midpoint or handle error [G]
        # This can happen if the cubic approximation fails to capture the crossing
        return 0.5 
        
    if len(valid_roots) == 1:
        return valid_roots[0]
    
    # Select the middle root if multiple roots exist
    valid_roots.sort()
    return valid_roots[len(valid_roots) // 2]

def solve_cubic_real(c0, c1, c2, c3):
    """
    Solves c3*x^3 + c2*x^2 + c1*x + c0 = 0 for real roots.
    Uses Cardano's method with robust sign handling for cube roots.
    """
    if c3 == 0:
        if c2 == 0:
            if c1 == 0:
                return []
            return [-c0 / c1]
        disc = c1**2 - 4 * c2 * c0
        if disc < 0:
            return []
        sqrt_disc = math.sqrt(disc)
        return [(-c1 - sqrt_disc) / (2 * c2), (-c1 + sqrt_disc) / (2 * c2)]
    
    a = c2 / c3
    b = c1 / c3
    c = c0 / c3
    
    p = b - a**2 / 3
    q = 2 * a**3 / 27 - a * b / 3 + c
    disc = q**2 / 4 + p**3 / 27
    
    roots = []
    if disc > 0:
        # One real root
        u_val = -q / 2 + math.sqrt(disc)
        v_val = -q / 2 - math.sqrt(disc)
        u = math.copysign(abs(u_val)**(1/3), u_val)
        v = math.copysign(abs(v_val)**(1/3), v_val)
        roots.append(u + v - a / 3)
    elif disc == 0:
        # Multiple real roots
        if q == 0:
            roots.append(-a / 3)
        else:
            u_val = -q / 2
            u = math.copysign(abs(u_val)**(1/3), u_val)
            roots.append(2 * u - a / 3)
            roots.append(-u - a / 3)
    else:
        # Three distinct real roots
        r = math.sqrt(-p**3 / 27)
        theta = math.acos(-q / (2 * r))
        m = 2 * r**(1/3)
        roots.append(m * math.cos(theta / 3) - a / 3)
        roots.append(m * math.cos((theta + 2 * math.pi) / 3) - a / 3)
        roots.append(m * math.cos((theta + 4 * math.pi) / 3) - a / 3)
        
    return roots
```

**Parameters:**
*   **Derivative Estimation:** Derivatives $f'(x_0)$ and $f'(x_1)$ are often not directly available and must be approximated using finite difference methods [S1]. The quality of these estimates significantly impacts accuracy [S1].

**Edge Cases and Pitfalls:**
*   **Multiple Roots:** A cubic polynomial can have up to three real roots [S2]. The location and count of these roots may be sensitive to small perturbations in function coefficients [S2].
*   **Root Selection:** If multiple roots exist, the "middle" root is selected to ensure a unique isovertex position [S2].
*   **Missing Roots:** If the cubic polynomial $p(x)$ does not have a root along the edge despite $f(x)$ having a zero-crossing, the interpolation fails to capture the crossing [S2]. This can occur if the derivative estimates are poor or the function is highly non-linear [S2].

## Subtopics

### Kochanek-Bartels Spline Interpolation
Kochanek-Bartels spline interpolation is an interpolation method used to generate smooth tour paths through a series of keyframes in Smokeview [S3]. Each interval is uniquely specified using a function value and a derivative at both endpoints, requiring four data values [S3]. Derivatives at endpoints are computed in terms of three parameters: bias, continuity, and tension [S3]. Each parameter ranges from -1 to 1 with a default value of 0, though bias and continuity are set to zero internally by Smokeview [S3]. A tension value of 1 results in a linear spline, removing smoothness [S3]. Unlike standard cubic splines, Kochanek-Bartels allows local control via bias, continuity, and tension parameters to adjust path smoothness or create abrupt changes [S3].

## Operating characteristics
Solving the 4x4 system is constant time $O(1)$ per edge due to the closed-form solution [S2]. Finding cubic roots is also $O(1)$ but involves more arithmetic operations than linear interpolation [G]. Memory complexity is $O(1)$ additional memory per edge [S2]. The method provides higher accuracy than linear interpolation, especially for non-linear implicit functions [S1, S2]. It is sensitive to derivative estimation errors [S1]. It may fail to find a root if the cubic approximation is poor [S2]. It should be used when high accuracy is required and derivative information (or good estimates) is available [S2].

## Background
Cubic Hermite Interpolation was proposed as a higher-order interpolation method to improve the accuracy of level crossing detection in isosurface extraction algorithms like Marching Cubes [S1, S2]. It builds on the concept of Hermite interpolation, which uses both function values and derivatives to construct a polynomial [S2]. The specific closed-form coefficients for the edge case are detailed in [S2], while the broader context of high-order interpolation methods is discussed in [S1].

## Sources
[S1] Estimation and Visualization of Isosurface Uncertainty from Linear and High-Order Interpolation Methods, https://arxiv.org/abs/2409.00043v1, fetched 2026-10-26
[S2] Accurate Isosurface Interpolation with Hermite Data, https://ieeexplore.ieee.org/document/7335492/, fetched 2026-11-05
[S3] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume I : User’s Guide, https://doi.org/10.6028/nist.sp.1017-1, fetched 2026-10-26
