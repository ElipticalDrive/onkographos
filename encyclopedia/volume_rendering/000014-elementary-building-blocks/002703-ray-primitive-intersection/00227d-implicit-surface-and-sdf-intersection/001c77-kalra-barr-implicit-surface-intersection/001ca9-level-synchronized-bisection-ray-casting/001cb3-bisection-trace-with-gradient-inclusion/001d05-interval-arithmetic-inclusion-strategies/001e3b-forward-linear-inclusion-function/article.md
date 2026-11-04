# Forward Linear Inclusion Function
Also called: Linear Bounding Function, Linear Inclusion Function

## Overview
The Forward Linear Inclusion Function computes a conservative interval bound for a scalar field $f$ along a ray, ensuring that the true function value lies within the calculated bounds for all points ahead of the current sample [S1]. It solves the problem of determining a safe maximum step size for sphere tracing by intersecting these linear bounds with the zero-level threshold [S1].

## How it works
**Inputs and Outputs**
*   **Input:** The current sample location on the ray $t_0$, the function value at that location $f(t_0)$, and the Lipschitz constant $\lambda$ for the function $f$ [S1].
*   **Output:** An interval $[f](t) = [f_-(t), f_+(t)]$ for $t > t_0$, and the computed step size $\Delta t$ to the next safe sample point [S1].

**Data Structures**
The method requires no complex data structures; it operates on scalar values for $t_0$, $f(t_0)$, and $\lambda$ [S1].

**Steps**
1.  **Evaluate Function:** Compute the scalar field value $v = f(t_0)$ at the current ray position $t_0$ [S1].
2.  **Define Bounds:** Construct the lower bound $f_-(t)$ and upper bound $f_+(t)$ as linear functions starting from $v$ at $t_0$ [S1]. The slope of the upper bound is $\lambda$ and the lower bound is $-\lambda$ [S1].
    $$ f_-(t) = f(t_0) - \lambda(t - t_0) $$
    $$ f_+(t) = f(t_0) + \lambda(t - t_0) $$
    This forms a bounding cone originating at $(t_0, f(t_0))$ [S1].
3.  **Compute Step Size:** Determine the distance to the zero-level set by finding the intersection of the bounding lines with the threshold [S1]. Since the bounds are symmetric around $f(t_0)$, the step size is determined by the magnitude of the function value divided by the Lipschitz constant:
    $$ \Delta t = \frac{|f(t_0)|}{\lambda} $$
    If $f(t_0) = 0$, the surface is hit exactly, and $\Delta t = 0$ [S1].
4.  **Advance Ray:** Update the ray position to $t_{\text{new}} = t_0 + \Delta t$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_linear_inclusion_step(f, t0, lambda_val):
    """
    Computes the next step size for sphere tracing using forward linear inclusion.
    
    Args:
        f: A callable that evaluates the scalar field f(t) at position t.
        t0: The current position on the ray (scalar).
        lambda_val: The Lipschitz constant for f (scalar, >= 0).
        
    Returns:
        delta_t: The safe step size to advance along the ray.
    """
    # Evaluate the scalar field at the current position
    f_t0 = f(t0)
    
    # Handle exact hit
    if f_t0 == 0.0:
        return 0.0
    
    # Compute step size: distance to zero crossing of the linear bound
    # The bound is f(t0) +/- lambda * (t - t0)
    # Intersection with 0: 0 = f(t0) + lambda * (t - t0) => t - t0 = -f(t0)/lambda
    # We take the absolute value for the forward step size
    delta_t = abs(f_t0) / lambda_val
    
    return delta_t
```

**Parameters**
*   $\lambda$: The Lipschitz constant, which must be an upper bound on the magnitude of the gradient of $f$ ($\|\nabla f\| \leq \lambda$) [G]. Typical values depend on the specific noise or implicit function being rendered [G]. If $\lambda$ is too small, the bound is invalid and the ray may tunnel through the surface [G].

**Edge Cases and Pitfalls**
*   **Zero Lipschitz Constant:** If $\lambda = 0$ and $f(t_0) \neq 0$, division by zero occurs [G]. Implementers should check for $\lambda < \epsilon$ and handle it as an error or infinite step [G].
*   **Inaccurate Lipschitz Constant:** The method assumes $\lambda$ is a valid global or local Lipschitz constant [G]. If the actual gradient exceeds $\lambda$, the inclusion property is violated, leading to visual artifacts (tunneling) [G].
*   **Precision:** For very small $|f(t_0)|$, floating-point precision errors in the division may result in tiny step sizes [G]. A minimum step size threshold may be applied to prevent infinite loops in flat regions [G].

## Subtopics

### [Improved Quadratic Taylor Inclusion Function](1902-improved-quadratic-taylor-inclusion-function.md)
Uses second-order Taylor expansion to provide tighter bounds for functions with significant curvature, switching to linear bounds when the quadratic derivative exceeds the first-order limit. Choose this when the scalar field has high curvature and $\lambda$ is large relative to the second derivative, allowing for larger steps than the linear method [G].

## Operating characteristics
**Time and Memory Complexity**
*   **Time:** $O(1)$ per step. The calculation involves one function evaluation, one absolute value, and one division [G].
*   **Memory:** $O(1)$. No additional memory is required beyond the scalar values [G].

**Strengths and Limitations**
*   **Strengths:** Simple to implement, computationally inexpensive, and provides a rigorous guarantee of no tunneling if $\lambda$ is valid [G].
*   **Limitations:** The bounds are often loose for functions with high curvature, leading to smaller step sizes than necessary [G]. It does not account for the second derivative of the function [G].

**When to Use**
*   Use when computational resources are limited or when the scalar field is relatively linear over the step size [G]. It is also useful as a fallback or comparison baseline for more complex inclusion functions [G].

## Background
The concept of forward inclusion functions was formalized in the context of sphere tracing for procedural volumetric noise [S1]. The specific formulation of the forward linear inclusion function is attributed to [AZ23] as cited in [S1]. It builds upon the general theory of interval arithmetic and Lipschitz continuity applied to ray marching [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
