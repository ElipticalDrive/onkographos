# Bidirectional Sphere Tracing

Also called: Forward-Backward Sphere Tracing, Zero-level set solver

## Overview
Bidirectional sphere tracing is a robust surface extraction technique that mitigates the convergence failures of standard sphere tracing when rays miss surface edges or intersect surfaces non-orthogonally [S1]. Unlike standard sphere tracing, which relies on a single forward march that may stall or overshoot, this method uses forward and backward traces to narrow the candidate range of the surface position, then refines the zero-level set using additional solvers [S1]. It is distinct from [FBM Sphere Tracing with BVH Acceleration](1728-fbm-sphere-tracing-with-bvh-acceleration.md) and [Coarse-to-Fine Sphere Tracing](4615-coarse-to-fine-sphere-tracing.md) by replacing the iterative step-size heuristic with a bounded search interval derived from two opposing traces.

## How it works
**Inputs and Outputs**
*   **Inputs**: Camera origin $\mathbf{r}_o$, ray direction $\mathbf{r}_d$ (normalized), Signed Distance Function $S(\mathbf{p})$, maximum ray distance $t_{max}$ (implied by rear intersection), forward iteration count $n=16$, convergence threshold $\epsilon_{conv}=5\times10^{-5}$, sample count $N_{samples}=100$, sectioning steps $N_{sec}=8$, gradient adjustment threshold $\epsilon_{grad}=0.005$, denominator clamp $\epsilon_{clamp}=0.01$ [S1].
*   **Outputs**: Refined zero-level set position $\hat{\mathbf{x}}_n$ for each ray, or a flag indicating no surface intersection [S1].

**Data Structures**
*   Ray state: $\mathbf{x}_f^i$ (current position in backward trace), $\mathbf{x}_n^i$ (current position in forward trace).
*   No persistent global data structures; the method is per-ray [S1].

**Algorithm Steps**

1.  **Forward Trace**:
    Initialize $\mathbf{x}_n^0 = \mathbf{r}_o$. Iterate $i$ from $0$ to $n-1$ (where $n=16$):
    $$ \mathbf{x}_n^{i+1} = \mathbf{x}_n^i + S(\mathbf{x}_n^i) \mathbf{r}_d $$
    [S1] This step advances the ray along the direction $\mathbf{r}_d$ by the distance indicated by the SDF.

2.  **Forward Convergence Check**:
    Let $\mathbf{x}_{nn} = \mathbf{x}_n^n$. If $|S(\mathbf{x}_{nn})| < 5\times10^{-5}$, the ray has converged. Set $\hat{\mathbf{x}}_n = \mathbf{x}_{nn}$ and terminate processing for this ray [S1].

3.  **Backward Trace** (for non-converged rays):
    Determine the rear intersection of the ray with a unit sphere to define the start point. Let $t_f$ be the rear intersection distance of $\mathbf{r}_d$ with the unit sphere centered at the origin (or appropriate reference frame).
    $$ \mathbf{x}_f^0 = \mathbf{r}_o + t_f \mathbf{r}_d $$
    [S1]
    Iterate $i$ from $0$ to $n-1$:
    $$ \mathbf{x}_f^{i+1} = \mathbf{x}_f^i - S(\mathbf{x}_f^i) \mathbf{r}_d $$
    [S1] Note: The source formula $x_f^{i+1} = x_0^i - S(x_f^i)r_d$ contains a likely typo ($x_0^i$); the logical interpretation consistent with sphere tracing is updating from the previous iteration's position $\mathbf{x}_f^i$ [S1]. Let $\mathbf{x}_{fn} = \mathbf{x}_f^n$.

4.  **Range Validation**:
    If $\mathbf{x}_{fn}$ is closer to the camera than $\mathbf{x}_{nn}$ (i.e., the distance along the ray for $\mathbf{x}_{fn}$ is less than that for $\mathbf{x}_{nn}$), conclude that no surface exists along this ray [S1].

5.  **Candidate Sampling**:
    Define the candidate range $[\mathbf{x}_{nn}, \mathbf{x}_{fn}]$. Evaluate $S(\mathbf{p})$ at $100$ uniformly spaced samples within this range [S1]. Search for the first zero-crossing of $S$ (change of sign) [S1].

6.  **Sectioning Refinement**:
    Assuming $S$ is locally monotonic within the identified interval, refine the zero-crossing location using $8$ steps of bisection (sectioning) [S1]. Let the result be $\hat{\mathbf{x}}_n$ [S1].

7.  **Gradient Adjustment** (Optional/Contextual):
    For points where $|S(\hat{\mathbf{x}}_n)| < 0.005$, apply a gradient-based adjustment to improve accuracy [S1].
    $$ \hat{\mathbf{x}}_n \leftarrow \hat{\mathbf{x}}_n - \frac{S(\hat{\mathbf{x}}_n)}{\nabla_{\mathbf{x}} S(\hat{\mathbf{x}}_n) \cdot \mathbf{r}_d} \mathbf{r}_d $$
    [S1]
    Guard against division by zero: only apply this adjustment if $|\nabla_{\mathbf{x}} S(\hat{\mathbf{x}}_n) \cdot \mathbf{r}_d| > 0.01$ [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bidirectional_sphere_tracing(r_o, r_d, S, n=16, eps_conv=5e-5, 
                                 n_samples=100, n_sec=8, eps_grad=0.005, 
                                 eps_clamp=0.01):
    """
    Performs bidirectional sphere tracing to find the zero-level set.
    
    Args:
        r_o: Camera origin (vector).
        r_d: Ray direction (normalized vector).
        S: Signed Distance Function callable S(p).
        n: Number of forward/backward iterations.
        eps_conv: Convergence threshold for early exit.
        n_samples: Number of samples in candidate range.
        n_sec: Number of bisection steps.
        eps_grad: Threshold for gradient adjustment.
        eps_clamp: Denominator clamp for gradient adjustment.
        
    Returns:
        x_hat: The refined zero-level set position, or None if no surface found.
    """
    
    # 1. Forward Trace
    x_n = r_o
    for i in range(n):
        dist = S(x_n)
        x_n = x_n + dist * r_d
        
    x_nn = x_n
    
    # 2. Forward Convergence Check
    if abs(S(x_nn)) < eps_conv:
        return x_nn
        
    # 3. Backward Trace
    # Calculate rear intersection with unit sphere.
    # Ray: r_o + t * r_d. Sphere: |p|^2 = 1.
    # |r_o + t*r_d|^2 = 1 => t^2 + 2(r_o.r_d)t + (|r_o|^2 - 1) = 0
    a = 1.0
    b = 2.0 * dot(r_o, r_d)
    c = dot(r_o, r_o) - 1.0
    discriminant = b*b - 4*a*c
    
    if discriminant < 0:
        # Ray does not intersect unit sphere. 
        # Fallback: use a large t_max or handle as no intersection.
        # Based on source, tf is rear intersection. 
        # If no intersection, this step is undefined in source. 
        # Assuming valid ray setup for typical usage.
        t_f = 1e6 # Placeholder for degenerate case
    else:
        sqrt_disc = sqrt(discriminant)
        t1 = (-b - sqrt_disc) / (2*a)
        t2 = (-b + sqrt_disc) / (2*a)
        # Rear intersection is the larger t
        t_f = max(t1, t2)
        
    x_f = r_o + t_f * r_d
    for i in range(n):
        dist = S(x_f)
        x_f = x_f - dist * r_d
        
    x_fn = x_f
    
    # 4. Range Validation
    # Check if x_fn is closer to camera than x_nn
    # Distance along ray: t_nn = |x_nn - r_o| (assuming r_o is origin or projected)
    # More robustly, compare t values if r_o is origin. 
    # General case: project onto r_d.
    t_nn = dot(x_nn - r_o, r_d)
    t_fn = dot(x_fn - r_o, r_d)
    
    if t_fn < t_nn:
        return None # No surface exists
        
    # 5. Candidate Sampling
    # Interpolate between x_nn and x_fn
    # We need to find the first zero crossing in [x_nn, x_fn]
    best_x = None
    min_t = infinity
    
    for i in range(n_samples + 1):
        t_frac = i / n_samples
        p = lerp(x_nn, x_fn, t_frac)
        val = S(p)
        
        # Check for zero crossing relative to previous sample
        if i > 0:
            prev_t_frac = (i - 1) / n_samples
            prev_p = lerp(x_nn, x_fn, prev_t_frac)
            prev_val = S(prev_p)
            
            if prev_val * val < 0:
                # Zero crossing found between prev_p and p
                # Store interval for sectioning
                interval_start = prev_p
                interval_end = p
                break
    else:
        # No zero crossing found in samples
        return None
        
    # 6. Sectioning Refinement
    a_sec = interval_start
    b_sec = interval_end
    for i in range(n_sec):
        mid = lerp(a_sec, b_sec, 0.5)
        val_mid = S(mid)
        val_a = S(a_sec)
        
        if val_a * val_mid < 0:
            b_sec = mid
        else:
            a_sec = mid
            
    x_hat = lerp(a_sec, b_sec, 0.5)
    
    # 7. Gradient Adjustment
    s_val = S(x_hat)
    if abs(s_val) < eps_grad:
        grad = compute_gradient(S, x_hat) # Numerical or analytical gradient
        denom = dot(grad, r_d)
        if abs(denom) > eps_clamp:
            x_hat = x_hat - (s_val / denom) * r_d
            
    return x_hat

def lerp(a, b, t):
    return a + t * (b - a)

def dot(v1, v2):
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def compute_gradient(S, p, eps=1e-4):
    # Central difference gradient
    dx = S(p + [eps, 0, 0]) - S(p - [eps, 0, 0])
    dy = S(p + [0, eps, 0]) - S(p - [0, eps, 0])
    dz = S(p + [0, 0, eps]) - S(p - [0, 0, eps])
    return [dx/(2*eps), dy/(2*eps), dz/(2*eps)]
```

## Operating characteristics
*   **Time Complexity**: $O(n + N_{samples} + N_{sec})$ per ray, where $n$ is the iteration count (16), $N_{samples}$ is 100, and $N_{sec}$ is 8. This is significantly higher than standard sphere tracing ($O(n)$) due to the backward trace and sampling phase [S1].
*   **Memory Complexity**: $O(1)$ per ray, as no global structures are required [S1].
*   **Strengths**: Robust against rays that miss surface edges or intersect surfaces non-orthogonally, where standard sphere tracing fails to converge [S1]. It guarantees a bounded search range [S1].
*   **Limitations**: Computationally expensive due to multiple SDF evaluations and the backward trace. Not naturally differentiable in its core search phase, requiring an extra forward step with gradient tracking for differentiable rendering [S1].
*   **When to use**: When high-quality surface extraction is required from an SDF, especially in complex geometries where standard marching might stall.
*   **When not to use**: When real-time performance is critical and approximate visual results are acceptable, or when the SDF is known to be well-behaved (orthogonal intersections).

## Background
Bidirectional sphere tracing was introduced by Yariv et al. in the context of Neural Lumigraph Rendering [S1, S60]. It was developed to address the convergence issues of standard sphere tracing when applied to neural implicit representations, where the SDF may not guarantee optimal step lengths for general shape configurations [S1]. The method adapts the sphere tracer to narrow a possible range of surface positions, which is then refined by additional solvers, ensuring robustness even when the nearest surface is not orthogonal to the ray [S1].

## Sources
[S1] Neural Lumigraph Rendering, https://doi.org/10.1109/cvpr46437.2021.00427, fetched 2026-10-26
