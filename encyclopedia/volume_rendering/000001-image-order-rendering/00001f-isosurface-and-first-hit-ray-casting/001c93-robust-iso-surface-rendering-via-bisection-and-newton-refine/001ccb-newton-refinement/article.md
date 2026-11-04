# Newton Refinement

Also called: Post-bisection refinement

## Overview
Newton Refinement is an optional post-processing step applied after the bisection phase of robust iso-surface rendering to achieve sub-pixel hit-point accuracy [S1, unofficial]. It improves the precision of the intersection point $t$ by iteratively correcting the position using directional derivatives, refining the result beyond the convergence threshold of the bisection method [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The current hit position $t$ (from bisection), ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, the implicit function $f(\mathbf{x})$, and its gradient $\nabla f(\mathbf{x})$ (or Lipschitz constant for bounds, but gradient is required for Newton steps) [S1, unofficial][S3].
*   **Output:** Refined hit position $t_{new}$ and corresponding 3D point $\mathbf{p}_{new}$ [S1, unofficial].

**Data Structures:**
*   `t`: Scalar float representing the ray parameter.
*   `origin`: Vector $(x, y, z)$ representing the ray origin.
*   `direction`: Normalized vector $(dx, dy, dz)$ representing the ray direction.
*   `hit`: Boolean flag indicating if the ray intersected the surface during bisection.

**Parameters:**
*   `newton_iter`: Number of Newton refinement iterations to perform. Default value is 3. Set to 0 to disable [S1, unofficial].

**Algorithm:**
1.  Check if the ray hit the surface (`hit == True`). If not, skip refinement [S1, unofficial].
2.  Initialize $t_{curr} = t$ (the result from bisection) [S1, unofficial].
3.  For $i$ from 0 to `newton_iter` - 1:
    a.  Compute the 3D position: $\mathbf{p} = \mathbf{o} + t_{curr} \cdot \mathbf{d}$ [S1, unofficial].
    b.  Evaluate the implicit function value: $f_p = f(\mathbf{p})$ [S1, unofficial].
    c.  Evaluate the gradient (directional derivative along ray is sufficient, but full gradient is standard): $\mathbf{g} = \nabla f(\mathbf{p})$ [S1, unofficial].
    d.  Compute the directional derivative: $f'_p = \mathbf{g} \cdot \mathbf{d}$ [S1, unofficial].
    e.  Check for division by zero or near-zero $f'_p$. If $|f'_p| < \epsilon$, break loop [G].
    f.  Update $t$: $t_{new} = t_{curr} - \frac{f_p}{f'_p}$ [S1, unofficial].
    g.  Clamp $t_{new}$ to the valid interval $[t_{lo}, t_{hi}]$ to prevent stepping outside the searched region [G].
    h.  Update $t_{curr} = t_{new}$ [S1, unofficial].
4.  Return $t_{curr}$ and $\mathbf{p}_{new} = \mathbf{o} + t_{curr} \cdot \mathbf{d}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def newton_refine(t, origin, direction, hit, t_lo, t_hi, newton_iter=3, f_func, grad_func):
    """
    Refine hit position t using Newton's method.
    
    Args:
        t: Current hit parameter from bisection.
        origin: Ray origin (3D vector).
        direction: Ray direction (normalized 3D vector).
        hit: Boolean, True if bisection found an intersection.
        t_lo: Lower bound of the bisection interval.
        t_hi: Upper bound of the bisection interval.
        newton_iter: Number of iterations (0 disables).
        f_func: Function f(p) -> float.
        grad_func: Function grad(p) -> (3D vector).
        
    Returns:
        Refined t and position p.
    """
    if not hit or newton_iter <= 0:
        p = origin + t * direction
        return t, p

    t_curr = t
    
    for _ in range(newton_iter):
        # Compute 3D position
        p = origin + t_curr * direction
        
        # Evaluate function and gradient
        f_val = f_func(p)
        grad = grad_func(p)
        
        # Directional derivative
        # Dot product of gradient and ray direction
        f_prime = sum(g * d for g, d in zip(grad, direction))
        
        # Guard against zero derivative (grazing ray or flat region)
        if abs(f_prime) < 1e-12:
            break
            
        # Newton step
        t_new = t_curr - f_val / f_prime
        
        # Clamp to bisection interval to ensure robustness
        t_new = max(t_lo, min(t_hi, t_new))
        
        t_curr = t_new

    p_final = origin + t_curr * direction
    return t_curr, p_final
```

**Edge Cases and Pitfalls:**
*   **Gradient Availability:** Newton refinement requires the gradient of the implicit function. If the gradient is not available or is zero, Newton's method may fail or diverge [S1, unofficial].
*   **Grazing Rays:** If the ray is nearly tangent to the surface, the directional derivative $f'_p$ approaches zero, causing numerical instability or large steps [G].
*   **Implementation Status:** In some provided code snippets (e.g., `render_robustness.py`), the feature is mentioned as available but not called in the core loop; the bisection result is used directly [S1, unofficial].

## Subtopics

### Predictive Newton Termination
A Newton iteration mode that terminates a vertex step if the predicted next error is below a tenth of the tolerance, without confirming with an evaluation [S2, unofficial]. The predicted error is calculated as $|f'| / (2 |f|) s^2$, where $f'$ is the gradient magnitude, $f$ is the function value, and $s$ is the step size [S2, unofficial]. This applies to exponentially decaying fields [S2, unofficial]. The normal used is that of the evaluation before the step [S2, unofficial]. On amide/mol datasets at $h=0.2$ and $0.12$, this reduces evaluations per vertex from 2.0-2.6 to 1.3-2.0, with a largest vertex residual of $1.4 \times 10^{-4} h$ (tolerance $10^{-4} h$) [S2, unofficial]. A bug existed where evaluators skipped a group of 64 vertices when the AND of `vdone` flags was set; a mix of 1 (converged) and 2 (predicted) cleared the AND, causing full runs after convergence. The fix ensures any nonzero flag counts [S2, unofficial].

### Ray Marching with Newton Localization
A ray tracing approach to find the first parameter along a ray where the field value equals the isovalue $c$, by isolating the root via sign change detection and localizing it precisely [S3]. The behavior is highly correlated to the chosen step size: small steps make the method computationally expensive, while larger step sizes can miss isosurface crossings [S3].

## Operating characteristics
*   **Time Complexity:** Adds $O(k)$ evaluations per hit, where $k$ is `newton_iter` (typically 3) [S1, unofficial].
*   **Memory Complexity:** $O(1)$ additional memory per ray [S1, unofficial].
*   **Performance:** Predictive termination can reduce evaluations per vertex by ~20-30% in specific GPU implementations [S2, unofficial].
*   **Strengths:** Achieves sub-pixel accuracy; leverages existing gradient information [S1, unofficial].
*   **Limitations:** Requires differentiable implicit functions; unstable for grazing rays or zero gradients [S1, unofficial].
*   **When to use:** When high-precision hit points are required and gradients are available [S1, unofficial].
*   **When not to use:** When gradients are unavailable or performance is critical and bisection precision is sufficient [S1, unofficial].

## Background
Newton Refinement is part of the "Robust Iso-surface Rendering via Bisection and Newton Refinement" framework, implemented in the `isoray` library [S1, unofficial]. It builds on the principle that bisection provides a robust, guaranteed interval, while Newton's method provides fast local convergence to higher precision [S1, unofficial]. The concept of combining ray marching for isolation with Newton localization for precision is also discussed in the context of scale-invariant integral surfaces [S3]. Predictive Newton termination was introduced in optimized GPU isosurface extraction code [S2, unofficial].

## Sources
[S1] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S2] md_gto_iso_gpu: optimized GPU isosurface extraction (non-blocking jobs, flood, one measured path), https://github.com/scanberg/mdlib/pull/15, fetched 2026-10-28 (unofficial)
[S3] Fast Ray Tracing of Scale‐Invariant Integral Surfaces, https://doi.org/10.1111/cgf.14208, fetched 2026-10-26
