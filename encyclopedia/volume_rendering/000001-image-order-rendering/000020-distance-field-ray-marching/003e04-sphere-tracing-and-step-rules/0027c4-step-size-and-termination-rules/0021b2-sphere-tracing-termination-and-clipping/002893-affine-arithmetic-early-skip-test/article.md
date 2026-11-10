# Affine Arithmetic Early Skip Test

Also called: Early exit Sphere Marching (ESM) condition, Underestimation test

## Overview
The Affine Arithmetic Early Skip Test is a constant-time heuristic used in distance-field ray marching to detect if a ray does not collide with an object within a specific working interval, thereby avoiding expensive root-finding operations [S1]. It distinguishes itself from sibling methods like Bracketed Newton Bisection (BNB) by providing immediate non-collision detection and a conservative starting point for subsequent searches, rather than searching for monotonic intervals of derivatives [S1].

## How it works
The method approximates the high-degree polynomial representation of the Signed Distance Field (SDF) as a quadratic equation to determine intersection existence [S1]. It relies on affine arithmetic principles to bound the uncertainty of the polynomial coefficients over a normalized working interval [S1].

### Inputs and Outputs
*   **Inputs**:
    *   Polynomial coefficients $cc_0, \dots, cc_9$ derived from the SDF expansion [S1].
    *   Ray entry point and direction (implicitly handled via the normalized parameter $t$) [S1].
    *   Boolean flag indicating if the ray starts inside the volume [S1].
*   **Outputs**:
    *   Boolean `skip`: True if the ray is guaranteed not to intersect the object within the working interval [S1].
    *   `t_start`: A candidate starting parameter for the subsequent root-finding search (if `skip` is False) [S1].

### Data Structures
The method operates on an array of scalar coefficients $cc_i$ and floating-point values [S1].

### Steps
1.  **Factorization**: Factorize the standard polynomial $P_N(\mathbf{p} + \mathbf{v}t)$ into the form $at^2 + bt + c = 0$ [S1]. The coefficients $b$ and $c$ are constant, while $a$ is a function of $t$ [S1]:
    $$
    b = cc_1
    $$
    $$
    c = cc_0
    $$
    $$
    a(t) = cc_9 t^7 + cc_8 t^6 + \dots + cc_3 t + cc_2
    $$
    [S1]

2.  **Define Working Interval**: Assume a normalized working range of $[0, 1]$ for the ray segment within the current node, where $t=0$ is the entry and $t=1$ is the exit [S1].

3.  **Compute Underestimated Coefficient $a$**: Calculate a conservative lower bound for $a(t)$ over $t \in [0, 1]$ using affine arithmetic underestimation [S1].
    *   If the ray starts **outside** the object (standard case), compute $a_{under}$ as:
        $$
        a_{under} = cc_2 + \sum_{i=3}^{9} cc_i \epsilon_i
        $$
        where $\epsilon_i$ is defined as:
        $$
        \epsilon_i = \begin{cases} 
        0 & \text{if } cc_i > 0 \\
        1 & \text{otherwise}
        \end{cases}
        $$
        [S1]
    *   If the ray starts **inside** the volume, swap the condition of $\epsilon_i$ (i.e., $\epsilon_i = 1$ if $cc_i > 0$, else $0$) to adapt the underestimation process for finding intersections from the inside [S1].

4.  **Solve Quadratic**: Solve the quadratic equation $a_{under} t^2 + b t + c = 0$ for $t$ [S1].
    *   Calculate the discriminant $\Delta = b^2 - 4 a_{under} c$ [G].
    *   If $\Delta < 0$, there are no real roots for the underestimated polynomial [G].

5.  **Skip Decision**:
    *   If the underestimated polynomial has no roots within the working interval $[0, 1]$ (either $\Delta < 0$ or roots are outside $[0, 1]$), then the original polynomial has no roots in this interval [S1]. Return `skip = True` [S1].
    *   If roots exist, return `skip = False` [S1].

6.  **Starting Point Selection**:
    *   If `skip` is False, identify the roots of the underestimated quadratic [G].
    *   Select the smaller positive root within the working interval as `t_start` [S1]. This value serves as a better starting point for the subsequent root-finding search using the original high-degree polynomial [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def affine_arithmetic_early_skip(cc, inside_volume=False):
    """
    Performs the Affine Arithmetic Early Skip Test.
    
    Args:
        cc: List of coefficients [cc_0, cc_1, ..., cc_9].
        inside_volume: Boolean, True if ray starts inside the volume.
        
    Returns:
        skip: Boolean, True if no intersection is possible in [0, 1].
        t_start: Float, candidate starting t for further search, or None.
    """
    # Step 1: Extract constant coefficients b and c
    b = cc[1]
    c = cc[0]
    
    # Step 3: Compute underestimated coefficient a
    # Base term is cc_2
    a_under = cc[2]
    
    # Summation for i from 3 to 9
    for i in range(3, 10):
        cc_i = cc[i]
        # Determine epsilon_i based on sign of cc_i
        if inside_volume:
            # Swap condition for inside volume
            epsilon_i = 1 if cc_i > 0 else 0
        else:
            # Standard condition for outside volume
            epsilon_i = 0 if cc_i > 0 else 1
        
        a_under += cc_i * epsilon_i
        
    # Step 4: Solve quadratic a_under * t^2 + b * t + c = 0
    discriminant = b * b - 4 * a_under * c
    
    if discriminant < 0:
        # No real roots for underestimated polynomial
        return True, None
        
    sqrt_disc = math.sqrt(discriminant)
    
    # Calculate roots
    # Avoid division by zero if a_under is 0, though typically a_under != 0 in valid SDF contexts
    if abs(a_under) < 1e-9:
        # Linear case: b*t + c = 0
        if abs(b) > 1e-9:
            t_linear = -c / b
            roots = [t_linear]
        else:
            roots = []
    else:
        t1 = (-b - sqrt_disc) / (2 * a_under)
        t2 = (-b + sqrt_disc) / (2 * a_under)
        roots = [t1, t2]
        
    # Step 5 & 6: Check roots within [0, 1] and select start point
    valid_roots = [t for t in roots if 0 <= t <= 1]
    
    if not valid_roots:
        # No roots in working interval -> Skip
        return True, None
    else:
        # Roots exist -> Cannot skip
        # Use the smaller root as starting point
        t_start = min(valid_roots)
        return False, t_start
```

### Edge Cases and Pitfalls
*   **Precision**: The method relies on floating-point arithmetic [S1]. If $a_{under}$ is very close to zero, the quadratic formula may suffer from cancellation errors [G]. A fallback to linear solving is recommended if $|a_{under}|$ is negligible [G].
*   **Conservatism**: The underestimation is conservative [S1]. If the underestimated polynomial has roots, the original polynomial *might* still not have roots (false positive for intersection) [S1]. However, if the underestimated polynomial has *no* roots, the original definitely has none (true negative for collision) [S1].
*   **Inside Volume**: The logic for $\epsilon_i$ must be correctly swapped when the ray origin is inside the volume; failure to do so results in an incorrect bound [S1].

## Operating characteristics
*   **Time Complexity**: $O(1)$ [S1]. The test involves a fixed number of arithmetic operations (summing 7 terms and solving a quadratic) regardless of the polynomial degree or scene complexity [S1].
*   **Memory Complexity**: $O(1)$ [S1]. No additional memory is allocated beyond local variables [S1].
*   **Strengths**: Provides constant-time detection for non-collisions, which is significantly faster than iterative root-finding methods like BNB when the ray misses the object [S1]. It also provides a heuristic starting point for the actual solver, potentially reducing the number of iterations required [S1].
*   **Limitations**: It is an approximation [S1]. It cannot find the exact intersection point, only determine existence or provide a starting guess [S1]. It is specific to the polynomial representation of the SDF [S1].
*   **When to use**: Use as a pre-check before invoking expensive root-finding algorithms (like BNB) in sphere tracing or analytic intersection solvers for high-order SDF approximations [S1].

## Background
The method was proposed in the context of rendering piecewise approximations of Signed Distance Fields (SDFs) using analytic intersections, specifically to address the lack of early skip strategies in methods like Bracketed Newton Bisection (BNB) [S1]. BNB searches monotonic intervals of derivatives, which can be computationally expensive and does not offer early exit for non-collisions [S1]. The Affine Arithmetic Early Skip Test was introduced to provide a constant-time alternative for detecting non-collisions and improving the convergence of subsequent root-finding steps [S1].

## Sources
[S1] Rendering piecewise approximations of SDFs through analytic intersections, https://doi.org/10.1016/j.cag.2024.103981, fetched 2026-10-27
