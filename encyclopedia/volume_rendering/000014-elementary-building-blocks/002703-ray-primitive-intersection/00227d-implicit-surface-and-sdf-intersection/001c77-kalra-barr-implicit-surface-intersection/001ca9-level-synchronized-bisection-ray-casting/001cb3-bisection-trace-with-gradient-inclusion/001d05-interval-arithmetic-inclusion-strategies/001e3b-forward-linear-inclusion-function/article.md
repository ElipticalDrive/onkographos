# Forward Linear Inclusion Function
Also called: Linear Bounding Function, Linear Inclusion Function

## Overview
The Forward Linear Inclusion Function is a method for computing valid, linear interval bounds for a scalar field function $f$ along a ray, used to determine safe step sizes in sphere tracing [S1]. It solves the problem of estimating the distance to the nearest surface intersection by bounding the function's variation using a Lipschitz constant $\lambda$. It sets itself apart from its siblings, such as the [Improved Quadratic Taylor Inclusion Function](1902-improved-quadratic-taylor-inclusion-function.md), by using first-order linear bounds rather than second-order quadratic approximations, making it computationally cheaper but potentially less tight for functions with significant curvature [S1].

## How it works
**Inputs and Outputs:**
- Input: The current sample location on the ray $t_0$, the function value at that location $f(t_0)$, and the Lipschitz constant $\lambda$ for the function $f$ [S1].
- Output: An interval $[f](t)$ for $t > t_0$, and the computed step size $\Delta t$ to the next sample point [S1].

**Data Structures:**
- No complex data structures are required. The method relies on scalar values for $t_0$, $f(t_0)$, and $\lambda$ [S1].

**Steps:**
1.  Evaluate the scalar field function $f$ at the current ray position $t_0$ to obtain $f(t_0)$ [S1].
2.  Construct the forward linear inclusion function $[f](t)$ for $t > t_0$. The function is defined as a pair of bounding lines starting at $f(t_0)$:
    $$ [f](t) = f(t_0) \pm \lambda(t - t_0) $$
    This ensures the inclusion property: $f(t_0) - \lambda(t - t_0) \leq f(t) \leq f(t_0) + \lambda(t - t_0)$ for all $t > t_0$ [S1].
3.  The lower bound $f_-(t) = f(t_0) - \lambda(t - t_0)$ and upper bound $f_+(t) = f(t_0) + \lambda(t - t_0)$ form a "bounding cone" starting at $f(t_0)$ [S1].
4.  Compute the step size $\Delta t$ by finding the intersection of the bounding lines with the zero axis (threshold $v=0$) [S1].
    -   For the lower bound: $0 = f(t_0) - \lambda(t - t_0) \implies t - t_0 = f(t_0)/\lambda$
    -   For the upper bound: $0 = f(t_0) + \lambda(t - t_0) \implies t - t_0 = -f(t_0)/\lambda$
    -   The safe step size is the absolute distance to the zero crossing:
        $$ \Delta t = \frac{|f(t_0)|}{\lambda} $$
    This step size guarantees that the next sample point is at least as far from the surface as the current bound allows, preventing overshooting [S1].

**Parameters:**
- $\lambda$: The Lipschitz constant of the function $f$. This value must be known or estimated (e.g., via [Lipschitz Constant Estimation](1572-lipschitz-constant-estimation.md)) [S1].
- $t_0$: The current position along the ray [S1].
- $f(t_0)$: The value of the implicit function at $t_0$ [S1].

**Edge Cases and Pitfalls:**
- If $\lambda = 0$, the step size is undefined. This implies a constant function, which has no surface intersection unless $f(t_0)=0$ [S1].
- If $f(t_0) = 0$, the step size is 0, indicating the ray is exactly on the surface [S1].
- The linear bound may be loose for functions with high curvature, leading to smaller step sizes than necessary and reduced performance compared to quadratic methods [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_linear_inclusion_step(t0, f_t0, lambda_val):
    """
    Computes the safe step size for sphere tracing using the forward linear inclusion function.
    
    Args:
        t0 (float): Current position on the ray.
        f_t0 (float): Value of the scalar field function at t0.
        lambda_val (float): Lipschitz constant of the function.
        
    Returns:
        float: The step size delta_t to the next sample point.
    """
    if lambda_val == 0:
        # Handle constant function case; no intersection or infinite step
        return 0.0
    
    # Step size is the distance to the zero crossing of the linear bound
    delta_t = abs(f_t0) / lambda_val
    
    return delta_t
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(1)$ per evaluation, as it involves only basic arithmetic operations [S1].
- **Memory Complexity:** $O(1)$, requiring only scalar storage for $t_0$, $f(t_0)$, and $\lambda$ [S1].
- **Performance:** Generally faster per step than quadratic methods due to simpler calculations, but may require more steps if the bounds are loose [S1].
- **Strengths:** Simple to implement, computationally inexpensive, and provides a safe upper bound on the step size [S1].
- **Limitations:** Less tight bounds for functions with significant curvature compared to quadratic inclusion functions, potentially leading to slower convergence [S1].
- **When to use:** When computational cost per step is critical and the function's curvature is low, or when a Lipschitz constant is readily available [S1].
- **When not to use:** When high accuracy or tight bounds are required for highly curved functions, where quadratic methods may be more efficient overall [S1].

## Background
The Forward Linear Inclusion Function is used to formalize step size calculation in sphere tracing by finding intersections of bounding functions with the threshold [S1]. It builds on the principles of interval arithmetic and Lipschitz continuity [S1]. The method is described in the context of fast sphere tracing of procedural volumetric noise [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
