# Forward Linear Inclusion Function
Also called: Linear Bounding Function, Linear Inclusion Function

## Overview
The Forward Linear Inclusion Function computes a valid, linear interval bound for a scalar field function $f$ along a ray, ensuring that the true function value at any point $t > t_0$ lies within the calculated interval. It solves the problem of determining a safe step size for sphere tracing by providing a conservative estimate of the function's growth rate. It sets itself apart from its siblings, such as the [Improved Quadratic Taylor Inclusion Function](1902-improved-quadratic-taylor-inclusion-function.md), by using first-order linear bounds rather than second-order quadratic ones, making it simpler but potentially less tight for functions with significant curvature.

## How it works
**Inputs and Outputs:**
- Input: The scalar field function $f$, a query point $t_0$ on the ray, the function value $f(t_0)$, and the Lipschitz constant $\lambda$ for the domain of interest.
- Output: An interval $[f](t) = [f_-(t), f_+(t)]$ for $t > t_0$, and the derived step size $\Delta t$.

**Data Structures:**
No complex data structures are required. The method relies on scalar values:
- $t_0$: Current sample position (float).
- $f_0$: Function value at $t_0$ (float).
- $\lambda$: Lipschitz constant (float, $\lambda \geq 0$).

**Formulas:**
The forward linear inclusion function is defined as a pair of linear bounding lines starting at $f(t_0)$:
$$ f_-(t) = f(t_0) - \lambda(t - t_0) $$
$$ f_+(t) = f(t_0) + \lambda(t - t_0) $$
Such that for all $t > t_0$:
$$ f_-(t) \leq f(t) \leq f_+(t) $$
The function is exact at $t_0$, meaning $f(t_0) = f_-(t_0) = f_+(t_0)$.

To determine the step size for sphere tracing, we solve for the intersection of the upper bound $f_+(t)$ with the zero threshold (assuming $f(t_0) > 0$):
$$ f(t_0) + \lambda(\Delta t) = 0 \implies \Delta t = -\frac{f(t_0)}{\lambda} $$
Since we seek a positive step size into the volume where $f$ decreases towards zero, and assuming $f(t_0) > 0$, the magnitude is:
$$ \Delta t = \frac{f(t_0)}{\lambda} $$
If $f(t_0) < 0$, the step size is similarly derived from the lower bound intersection. In general, the step size is:
$$ \Delta t = \frac{|f(t_0)|}{\lambda} $$

**Parameters:**
- $\lambda$: The Lipschitz constant. This must be an upper bound on the gradient magnitude $|\nabla f|$ over the region of interest. Typical values depend on the specific implicit function (e.g., for noise functions, this is often precomputed or estimated via [Lipschitz Constant Estimation](1572-lipschitz-constant-estimation.md)).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_linear_inclusion_step(f, t0, lambda_val):
    """
    Computes the safe step size for sphere tracing using the forward linear inclusion function.
    
    Args:
        f: A callable that evaluates the scalar field f(t) at position t.
        t0: The current sample position on the ray.
        lambda_val: The Lipschitz constant (upper bound on |gradient|).
        
    Returns:
        delta_t: The maximum safe step size.
    """
    # Evaluate the function at the current point
    f0 = f(t0)
    
    # Guard against division by zero if the function is constant
    if lambda_val == 0:
        # If the gradient bound is zero, the function is constant.
        # If f0 is not zero, we never intersect (or are always on surface).
        # Return a minimal step or handle as special case.
        # [G] In practice, if lambda is 0 and f0 != 0, no intersection exists.
        return 0.0 if f0 != 0 else float('inf')
        
    # Calculate step size based on the linear bound intersection
    # delta_t = |f(t0)| / lambda
    delta_t = abs(f0) / lambda_val
    
    return delta_t
```

**Edge Cases and Pitfalls:**
- **Zero Lipschitz Constant:** If $\lambda = 0$, the function is constant. If $f(t_0) \neq 0$, there is no intersection. The step size should be handled to avoid division by zero.
- **Tightness:** If the function has high curvature, the linear bound may be significantly looser than the actual function, leading to smaller step sizes than necessary and reduced performance. This is the primary motivation for using quadratic variants like the [Improved Quadratic Taylor Inclusion Function](1902-improved-quadratic-taylor-inclusion-function.md).
- **Lipschitz Constant Accuracy:** Using an overly large $\lambda$ results in conservative (small) step sizes. Using an $\lambda$ that is too small violates the inclusion property, potentially causing the ray to skip the surface.

## Subtopics

### [Improved Quadratic Taylor Inclusion Function](1902-improved-quadratic-taylor-inclusion-function.md)
Uses second-order Taylor expansion to provide tighter bounds for functions with significant curvature, switching to linear bounds when the quadratic derivative exceeds the first-order limit. Choose this when the scalar field has high curvature and the linear bound is too loose.

## Operating characteristics
- **Time Complexity:** $O(1)$ per step, involving one function evaluation and basic arithmetic.
- **Memory Complexity:** $O(1)$, requiring only scalar storage for $t_0$, $f(t_0)$, and $\lambda$.
- **Performance:** Generally faster per step than quadratic methods due to simpler arithmetic, but may require more steps if the bounds are loose.
- **Strengths:** Simple to implement, robust, and provides a guaranteed safe step size if $\lambda$ is valid.
- **Limitations:** Bounds can be loose for highly curved functions, leading to inefficient sampling. Requires a valid Lipschitz constant.
- **When to use:** When the scalar field is relatively smooth or linear, or when computational simplicity is prioritized over step size optimality.
- **When not to use:** When the scalar field has high curvature and the linear bound results in significantly smaller step sizes than the quadratic bound, causing performance issues.

## Background
The concept of inclusion functions for interval arithmetic was formalized in the context of ray tracing and sphere tracing to provide rigorous bounds for step size calculation. The forward linear inclusion function specifically adapts the classic sphere tracing step size equation $\Delta t = f(t_0)/\lambda$ into a formal interval arithmetic framework, ensuring that the step size is always safe with respect to the Lipschitz constant. This approach is detailed in [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
