# Quadratic Taylor Inclusion Functions

Also called: 2nd order Taylor bounds, Naive Quadratic Bounds, Quadratic bounds, Regular Quad. Bounds, Regular Quadratic Taylor Bounds

## Overview
Quadratic Taylor Inclusion Functions (QTIF) are a method for adapting step size in sphere tracing by using quadratic forward inclusion bounds instead of linear ones, explicitly taking the ray direction into account [S1]. They solve the problem of inefficient step sizes for grazing rays by providing tighter bounds when the function derivative is Lipschitz continuous and computable on the fly [S1]. Unlike linear inclusion functions, which only bound the function value based on the first derivative, QTIFs bound the function value based on the first derivative and the second-order Lipschitz constant of that derivative, allowing for larger steps in regions of high curvature [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Current position on the ray $t_0$, the implicit field function $F_n(t)$ (sum of $n$ bounded functions), the first derivative $F_n'(t_0)$, and the second-order Lipschitz constant $\gamma_{F_n}$.
*   **Output:** A quadratic interval $[F_n](t)$ bounding the function value for $t > t_0$, used to compute the maximum safe step size to the isosurface (typically $F_n(t)=0$).

**Data Structures:**
*   **Lipschitz Constants:** For a sum of functions $F_n = \sum_{m=1}^n f_m$, the second-order Lipschitz constant is the sum of individual constants:
    $$ \gamma_{F_n} = \sum_{m=1}^n \gamma_{f_m} $$
    [S1]
*   **Derivative:** The derivative $f'(t)$ denotes the directional derivative in the ray direction $\mathbf{u}$:
    $$ f'(t) = \mathbf{u} \cdot \nabla f(\boldsymbol{\delta}(t)) $$
    [S1]

**Formulas:**
The forward quadratic Taylor inclusion function for $F_n$ from position $t_0$ is defined as:
$$ [F_n](t) = F_n(t_0) + (t - t_0)\left(F_n'(t_0) \pm \frac{1}{2}\gamma_{F_n} \cdot (t - t_0)\right) $$
[S1]

To find the step size $h$ to the isosurface (where $[F_n](t_0 + h) = 0$), one solves the quadratic equation derived from the tighter bound (assuming the function decreases towards the surface):
$$ F_n(t_0) + h F_n'(t_0) - \frac{1}{2}\gamma_{F_n} h^2 = 0 $$
[S1]

**Numbered Steps:**
1.  Compute the current field value $F_n(t_0)$ and its directional derivative $F_n'(t_0)$ at the current ray position $t_0$ [S1].
2.  Compute or retrieve the second-order Lipschitz constant $\gamma_{F_n}$ for the current set of active functions [S1].
3.  Construct the quadratic inclusion bound $[F_n](t)$ using the formula above [S1].
4.  Solve for the root $h$ of the equation $[F_n](t_0 + h) = 0$. This yields the maximum safe step size [S1].
5.  Advance the ray by $h$ [S1].
6.  **Sub-steps (Optional):** If sub-stepping is used, estimate the derivative at the new location $t_1 = t_0 + step_i$ using the worst-case scenario for the second-order Lipschitz constant:
    $$ F'_i(t_1) = F'_i(t_0) + (\gamma_i \cdot step_i) $$
    Then add the next derivative $f'_{i+1}(t_1)$ to compute the new quadratic bound for the next iteration [S1].

**Parameters:**
*   $\gamma$: The second-order Lipschitz constant of the derivative. For FBM, this grows exponentially with the number of octaves $n$ as $\gamma_{noise} \cdot (2^n - 1)$ [S1].

**Edge Cases and Pitfalls:**
*   **FBM Inefficiency:** For Fractal Brownian Motion (FBM), the second-order Lipschitz constant $\gamma_{F_n}$ grows exponentially with the number of octaves $n$. This makes regular quadratic bounds extremely inefficient, often performing worse than linear bounds because the quadratic term grows too fast, leading to smaller step sizes and more iterations [S1].
*   **Derivative Cost:** Requires computing $f'(t)$ at each step, which can be costly or invalid depending on the function $f$ [S1].
*   **Lipschitz Continuity:** Requires $f'(t)$ to be Lipschitz continuous. If this condition is not met, the inclusion function is invalid [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def quadratic_taylor_step(t0, F_n, F_prime_n, gamma_Fn, ray_direction):
    """
    Computes the next step size using Quadratic Taylor Inclusion Functions.
    
    Args:
        t0: Current position on the ray.
        F_n: Value of the implicit function at t0.
        F_prime_n: Directional derivative of the implicit function at t0.
        gamma_Fn: Second-order Lipschitz constant of the derivative.
        ray_direction: Unit vector of the ray.
        
    Returns:
        step_size: The maximum safe step size.
    """
    # Solve F_n(t0) + h * F_prime_n - 0.5 * gamma_Fn * h^2 = 0
    # Rearranging: 0.5 * gamma_Fn * h^2 - F_prime_n * h - F_n(t0) = 0
    # Using quadratic formula: h = (b +/- sqrt(b^2 - 4ac)) / 2a
    # a = 0.5 * gamma_Fn
    # b = -F_prime_n
    # c = -F_n(t0)
    
    a = 0.5 * gamma_Fn
    b = -F_prime_n
    c = -F_n
    
    # Discriminant
    discriminant = b**2 - 4 * a * c
    
    if discriminant < 0:
        # No real root within the bound, take a small safe step [G]
        return 1e-6
        
    sqrt_disc = math.sqrt(discriminant)
    
    # We want the smallest positive step size
    # h1 = (F_prime_n + sqrt_disc) / gamma_Fn
    # h2 = (F_prime_n - sqrt_disc) / gamma_Fn
    
    # Since a > 0, the parabola opens upward. 
    # The bound is F_n(t0) + h*F' - 0.5*gamma*h^2.
    # We assume F_n decreases towards the surface, so F_prime is likely negative.
    # We take the root that corresponds to the first intersection.
    
    h1 = (F_prime_n + sqrt_disc) / gamma_Fn
    h2 = (F_prime_n - sqrt_disc) / gamma_Fn
    
    # Select the smallest positive step
    if h1 > 0 and h2 > 0:
        step_size = min(h1, h2)
    elif h1 > 0:
        step_size = h1
    elif h2 > 0:
        step_size = h2
    else:
        # Both negative, meaning the surface is behind or bound is invalid [G]
        return 1e-6
        
    return step_size
```

## Subtopics

### Taylor Polynomial Approximation for Root Isolation
A method for arbitrary field definitions using Taylor polynomial approximations to perform root isolation [S2]. It defines adaptive intervals along the ray using a gradient-based heuristic, analyzes each interval using two Taylor approximations of degree two defined from both end-points, and subdivides intervals until a single root is found, followed by a bounded Newton method [S2]. It requires computation of higher-order derivatives and a well-chosen initial step-size, which is inefficient in the context of sphere tracing [S2].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Higher per-step cost than linear bounds due to the need to compute the first derivative $f'(t)$ and the second-order Lipschitz constant $\gamma$ [S1].
*   **Memory:** No significant additional memory overhead compared to linear bounds, assuming $\gamma$ is precomputed or computed on the fly [S1].

**Measured Performance:**
*   On the "Sponge" scene without enhancements: Regular Quad bounds have 599.9 avg steps, 8969.0 avg octaves, and 181.8 ms time, resulting in a $\times 0.6$ speedup (slowdown) compared to Linear bounds [S1].
*   On the "Sponge (far)" scene without enhancements: Regular Quad bounds have 416.6 avg steps, 4671.5 avg octaves, and 96.2 ms time, resulting in a $\times 0.4$ speedup (slowdown) compared to Linear bounds [S1].
*   When combined with BVH enhancements (Early stop+1, sub-stepping), quadratic bounds can achieve $\times 4.0$ speedup over linear bounds with the same enhancements [S1].

**Strengths and Limitations:**
*   **Strengths:** Provides tighter bounds for grazing rays when the function derivative is Lipschitz continuous, potentially reducing the number of steps in smooth regions [S1].
*   **Limitations:** For FBM, the exponential growth of $\gamma_{F_n}$ with the number of octaves makes regular quadratic bounds extremely inefficient and often worse than linear bounds [S1]. The overhead of computing gradients can negate benefits if not optimized [S1].

**When to Use:**
*   Use when the function derivative is Lipschitz continuous and computable, and when the scene does not involve high-frequency FBM with many octaves without BVH enhancements [S1].
*   Use with BVH enhancements (like Early stop+1) to mitigate the cost of computing $\gamma$ for all octaves [S1].

**When Not to Use:**
*   Do not use regular quadratic bounds directly on high-octave FBM without enhancements, as they perform worse than linear bounds [S1].
*   Do not use if the derivative $f'(t)$ is not computable or not Lipschitz continuous [S1].

## Background
The concept of Lipschitz continuity was first used by Kalra and Barr [KB89] for guaranteed localization of intersections between a ray and an implicit surface, using Lipschitz bounds of both the field function and its gradient [S2]. The specific quadratic Taylor inclusion function for sphere tracing was introduced by Aydinlilar et al. [AZ23] as an improvement over linear inclusion functions to better handle grazing rays [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
[S2] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
