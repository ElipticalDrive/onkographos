# Forward Linear Inclusion Function

Also called: Linear Bounding Function, Linear Inclusion Function

## Overview
The Forward Linear Inclusion Function is a method for computing valid, linear bounds on a scalar field function $f(t)$ along a ray, used to determine safe step sizes in sphere tracing [S1]. It solves the problem of finding the maximum distance to the next surface intersection by constructing a bounding cone that is guaranteed to contain the true function values [S1]. It sets itself apart from its siblings by relying solely on the function value and a global Lipschitz constant, offering black-box compatibility at the cost of potentially looser bounds compared to gradient-based or higher-order methods [S1, 1410-interval-arithmetic-inclusion-strategies.md].

## How it works
**Inputs and outputs:**
- Input: The scalar field function $f: \mathbb{R} \rightarrow \mathbb{R}$ evaluated at the current ray position $t_0$, denoted $f(t_0)$; the global Lipschitz constant $\lambda$ for the field; the current ray parameter $t_0$.
- Output: A step size $\Delta t$ representing the maximum safe distance to the next zero-crossing, and the inclusion bounds $[f_-(t), f_+(t)]$ for $t > t_0$.

**Data structures:**
- No complex data structures are required. The method operates on scalar values.

**Numbered steps:**
1. Evaluate the scalar field at the current ray position $t_0$ to obtain $f(t_0)$ [S1].
2. Define the forward linear inclusion function $[f](t)$ for $t > t_0$ as a pair of linear bounding lines starting at $(t_0, f(t_0))$ [S1].
3. The lower bound $f_-(t)$ and upper bound $f_+(t)$ are defined as:
   $$ f_-(t) = f(t_0) - \lambda(t - t_0) $$
   $$ f_+(t) = f(t_0) + \lambda(t - t_0) $$
   where $\lambda$ is the Lipschitz constant such that $|f(t) - f(t_0)| \leq \lambda |t - t_0|$ for all $t$ [S1].
4. To find the step size, solve for the intersection of the bounding lines with the zero threshold ($v=0$) [S1]. Since the function value $f(t_0)$ indicates the distance to the surface in normalized space, the step size is determined by the intersection of the bound with zero [S1].
5. The step size $\Delta t$ is calculated as:
   $$ \Delta t = \frac{|f(t_0)|}{\lambda} $$
   The sign of $f(t_0)$ determines the direction of the surface relative to the ray, but the step size is always positive [S1].

**Formulas:**
- Inclusion bounds: $[f](t) = f(t_0) \pm \lambda(t - t_0)$ [S1].
- Step size: $\Delta t = f(t_0) / \lambda$ (assuming $f(t_0) > 0$ for exterior points) [S1].

**Parameters:**
- $\lambda$: The Lipschitz constant of the scalar field. This must be known or estimated beforehand. Typical values depend on the specific noise function or implicit surface being rendered.

**Edge cases and pitfalls:**
- If $f(t_0) = 0$, the step size is 0, indicating the ray is exactly on the surface [S1].
- If $\lambda = 0$, the function is constant, and no intersection can be found unless $f(t_0) = 0$ [G].
- The bound is only valid if $\lambda$ is a true global Lipschitz constant for the field. Underestimating $\lambda$ leads to invalid bounds and missed intersections [S1].
- The linear bound may be loose for functions with high curvature, leading to smaller step sizes than necessary [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_linear_inclusion_step(f, t0, lambda_val):
    """
    Calculate the step size for sphere tracing using the Forward Linear Inclusion Function.
    
    Args:
        f (callable): The scalar field function f(t).
        t0 (float): The current ray parameter.
        lambda_val (float): The global Lipschitz constant of the field.
        
    Returns:
        float: The step size to advance the ray.
    """
    # Evaluate the function at the current position
    f_t0 = f(t0)
    
    # Handle the case where we are exactly on the surface
    if abs(f_t0) < 1e-6:
        return 0.0
    
    # Calculate step size using the linear inclusion bound intersection with zero
    # The bound is f(t0) +/- lambda * (t - t0).
    # Intersection with 0: 0 = f(t0) + lambda * (t - t0) => t - t0 = -f(t0)/lambda
    # We take the absolute value for the step size
    step = abs(f_t0) / lambda_val
    
    return step
```

## Subtopics

### [Improved Quadratic Taylor Inclusion Function](1902-improved-quadratic-taylor-inclusion-function.md)
The Improved Quadratic Taylor Inclusion Function is a method for computing tighter, valid bounds on a scalar field function $f(t)$ along a ray, used to determine safe step sizes in sphere tracing.

## Operating characteristics
- **Time complexity:** $O(1)$ per step, as it requires only one function evaluation and basic arithmetic [S1].
- **Memory complexity:** $O(1)$, requiring no additional storage beyond the current state [S1].
- **Measured performance:** Performance depends heavily on the tightness of $\lambda$. A loose $\lambda$ results in smaller step sizes and more evaluations, reducing performance [S1].
- **Scaling:** Scales linearly with the number of steps required to reach the surface [S1].
- **Strengths:** Simple to implement, requires no gradient information, works with black-box functions [S1].
- **Limitations:** Can produce loose bounds for functions with high curvature, leading to inefficient step sizes compared to quadratic or gradient-based methods [S1].
- **When to use:** When gradient information is unavailable or too expensive to compute, and the Lipschitz constant is known [S1].
- **When not to use:** When high performance is critical and the function has significant curvature, making linear bounds too loose [S1].

## Background
The Forward Linear Inclusion Function is based on the principles of interval arithmetic and Lipschitz continuity applied to ray tracing [S1]. It formalizes the step size calculation in sphere tracing by finding intersections of bounding functions with the threshold [S1]. The method is described in the context of fast sphere tracing of procedural volumetric noise [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
