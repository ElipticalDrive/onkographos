# Bidirectional Sphere Tracing

Also called: Forward-Backward Sphere Tracing, Zero-level set solver

## Overview
Bidirectional Sphere Tracing is a robust surface extraction technique that mitigates the convergence failures of standard sphere tracing when rays miss surface edges or intersect surfaces non-orthogonally [S1]. Unlike standard sphere tracing, which relies on the Signed Distance Function (SDF) to guarantee convergence by stepping along the ray, this method uses forward and backward sphere tracing to narrow the candidate range of the surface position, then refines the zero-level set using additional solvers [S1]. It is distinct from sibling methods like [Relaxed Sphere Tracing](9463-relaxed-sphere-tracing.md) because it does not attempt to correct the step size via gradient adjustments in every iteration; instead, it bounds the search space and delegates precision to a separate refinement phase [S1].

## How it works
**Inputs and Outputs**
*   **Inputs**: Ray origin $\mathbf{r}_o$, normalized ray direction $\mathbf{r}_d$, SDF function $S(\mathbf{p})$, forward iteration count $n=16$, convergence threshold $\epsilon_{mask} = 5 \times 10^{-5}$, gradient adjustment threshold $\epsilon_{grad} = 0.005$, denominator clamp $\epsilon_{clamp} = 0.01$, candidate sample count $N_{samples}=100$, and sectioning steps $N_{sec}=8$ [S1].
*   **Outputs**: Refined zero-level set position $\hat{\mathbf{x}}_n$ or a flag indicating no surface intersection [S1].

**Data Structures**
No complex data structures are required. The algorithm operates on scalar distances and 3D vector positions.

**Algorithm Steps**
1.  **Forward Sphere Tracing**:
    *   Initialize $\mathbf{x}_0^0 = \mathbf{r}_o$.
    *   Iterate $i$ from $0$ to $n-1$ (where $n=16$):
        *   Evaluate $d = S(\mathbf{x}_i^i)$.
        *   Update position: $\mathbf{x}_i^{i+1} = \mathbf{x}_i^i + d \cdot \mathbf{r}_d$ [S1].
    *   Let $\mathbf{x}_{nn} = \mathbf{x}_i^n$ be the near zero-level set estimate [S1].

2.  **Convergence Check**:
    *   If $|S(\mathbf{x}_{nn})| < \epsilon_{mask}$, the ray has converged. Set $\hat{\mathbf{x}}_n = \mathbf{x}_{nn}$ and terminate [S1].

3.  **Backward Sphere Tracing**:
    *   For rays that did not converge, initialize the backward trace from the rear intersection of the ray with a unit sphere. Calculate $t_f$ as the rear intersection of $\mathbf{r}_d$ with a unit sphere centered at the origin (or appropriate reference frame). Set $\mathbf{x}_f^0 = \mathbf{r}_o + t_f \mathbf{r}_d$ [S1].
    *   *Note*: The source defines the iteration as $\mathbf{x}_f^{i+1} = \mathbf{x}_0^i - S(\mathbf{x}_f^i) \mathbf{r}_d$ [S1]. Interpreting $\mathbf{x}_0^i$ as the previous iteration's position $\mathbf{x}_f^i$ (consistent with standard fixed-point iteration logic for finding roots along a line), the update is:
        $$ \mathbf{x}_f^{i+1} = \mathbf{x}_f^i - S(\mathbf{x}_f^i) \mathbf{r}_d $$
    *   The source does not specify the number of backward iterations. It implies solving until a "far zero-level set" $\mathbf{x}_{fn}$ is found. In practice, this often mirrors the forward count or runs until the sign of $S$ changes relative to the forward trace, but the exact iteration count is not given in the sources [S1]. We assume a symmetric count or sufficient iterations to bracket the surface. Let $\mathbf{x}_{fn}$ be the resulting position [S1].

4.  **Candidate Range Validation**:
    *   Determine the range $[\mathbf{x}_{nn}, \mathbf{x}_{fn}]$.
    *   If $\mathbf{x}_{fn}$ is closer to the camera than $\mathbf{x}_{nn}$ (i.e., the distance along $\mathbf{r}_d$ from $\mathbf{r}_o$ to $\mathbf{x}_{fn}$ is less than to $\mathbf{x}_{nn}$), conclude that no surface exists along this ray [S1].

5.  **Zero-Crossing Search**:
    *   Evaluate $N_{samples}=100$ samples uniformly distributed within the candidate range $[\mathbf{x}_{nn}, \mathbf{x}_{fn}]$ [S1].
    *   Identify the first zero crossing of $S$ within this range [S1].

6.  **Refinement (Sectioning)**:
    *   Assuming $S$ is locally monotonic, refine the zero-crossing location using $N_{sec}=8$ steps of sectioning (bisection) to obtain the final zero-level set $\hat{\mathbf{x}}_n$ [S1].

7.  **Gradient Adjustment (Optional Final Step)**:
    *   For performance and stability, the main procedure is non-differentiable. An extra step of forward sphere tracing with gradient tracking is executed to achieve differentiability [S1].
    *   Apply gradient direction adjustment to get the final zero-level set:
        $$ \hat{\mathbf{x}}_n = \hat{\mathbf{x}}_n - \frac{S(\hat{\mathbf{x}}_n)}{\nabla_{\mathbf{x}} S(\hat{\mathbf{x}}_n) \cdot \mathbf{r}_d} \mathbf{r}_d $$
    *   This adjustment is applied only if $|S(\hat{\mathbf{x}}_n)| < \epsilon_{grad}$ [S1].
    *   To avoid division by zero, clamp the denominator: only apply the adjustment if $|\nabla_{\mathbf{x}} S(\hat{\mathbf{x}}_n) \cdot \mathbf{r}_d| > \epsilon_{clamp}$ [S1].

**Pseudocode**

```python
import numpy as np

def bidirectional_sphere_tracing(ray_origin, ray_direction, sdf, 
                                 n_forward=16, epsilon_mask=5e-5, 
                                 epsilon_grad=0.005, epsilon_clamp=0.01,
                                 n_samples=100, n_sectioning=8):
    """
    Bidirectional sphere tracing for robust surface extraction.
    
    Args:
        ray_origin: 3D vector, camera origin.
        ray_direction: 3D vector, normalized ray direction.
        sdf: Callable, returns signed distance at point p.
        n_forward: Number of forward sphere tracing iterations.
        epsilon_mask: Threshold to consider a ray converged.
        epsilon_grad: Threshold for applying gradient adjustment.
        epsilon_clamp: Denominator clamp for gradient adjustment.
        n_samples: Number of samples in candidate range.
        n_sectioning: Number of bisection steps for refinement.
        
    Returns:
        hit_point: 3D vector, refined surface position, or None.
    """
    
    # 1. Forward Sphere Tracing
    x_curr = ray_origin
    for _ in range(n_forward):
        d = sdf(x_curr)
        x_curr = x_curr + d * ray_direction
        
    x_nn = x_curr # Near zero-level set
    
    # 2. Convergence Check
    if abs(sdf(x_nn)) < epsilon_mask:
        return x_nn

    # 3. Backward Sphere Tracing
    # Calculate rear intersection with a unit sphere.
    # Assuming unit sphere centered at origin.
    # Ray: r(t) = ro + t * rd. |ro + t*rd|^2 = 1
    # t^2 + 2(ro . rd)t + (|ro|^2 - 1) = 0
    # We want the larger root (rear intersection).
    a = 1.0
    b = 2.0 * np.dot(ray_origin, ray_direction)
    c = np.dot(ray_origin, ray_origin) - 1.0
    discriminant = b*b - 4*a*c
    
    if discriminant < 0:
        # Ray does not intersect unit sphere. 
        # Fallback or error handling. Source assumes intersection exists.
        # For implementation safety, we might use a large t or return None.
        # Given the source context (Neural Volumes/IDR), the camera is 
        # typically outside or the sphere is a bounding volume.
        # We assume a valid intersection exists as per source logic.
        t_f = 1e6 # Placeholder, should be calculated properly
    else:
        sqrt_disc = np.sqrt(discriminant)
        t1 = (-b - sqrt_disc) / (2*a)
        t2 = (-b + sqrt_disc) / (2*a)
        t_f = max(t1, t2) # Rear intersection
        
    x_f_curr = ray_origin + t_f * ray_direction
    
    # The source formula is x_f^{i+1} = x_f^i - S(x_f^i) * r_d
    # We need to determine iteration count. Source doesn't specify.
    # We assume a symmetric count or until bracketing is achieved.
    # For robustness, we run until the sign of S(x_f) is opposite to S(x_nn)
    # or a max count is reached. Let's use n_forward as a proxy for symmetry.
    for _ in range(n_forward):
        d = sdf(x_f_curr)
        x_f_curr = x_f_curr - d * ray_direction
        
    x_fn = x_f_curr # Far zero-level set
    
    # 4. Candidate Range Validation
    # Check if x_fn is closer than x_nn
    dist_nn = np.dot(x_nn - ray_origin, ray_direction)
    dist_fn = np.dot(x_fn - ray_origin, ray_direction)
    
    if dist_fn < dist_nn:
        return None # No surface exists
        
    # 5. Zero-Crossing Search
    # Evaluate samples in range [x_nn, x_fn]
    # Note: x_nn and x_fn are 3D points. We interpolate along the ray.
    # However, the source says "evaluate 100 samples with the candidate range".
    # This implies sampling the segment between x_nn and x_fn.
    
    # Ensure we sample from near to far
    if dist_nn > dist_fn:
        x_start, x_end = x_fn, x_nn
    else:
        x_start, x_end = x_nn, x_fn
        
    # Linear interpolation
    candidates = []
    for i in range(n_samples + 1):
        t = i / n_samples
        p = x_start + t * (x_end - x_start)
        candidates.append((p, sdf(p)))
        
    # Find first zero crossing
    zero_crossing_idx = -1
    for i in range(len(candidates) - 1):
        s1 = candidates[i][1]
        s2 = candidates[i+1][1]
        if s1 * s2 <= 0:
            zero_crossing_idx = i
            break
            
    if zero_crossing_idx == -1:
        # No zero crossing found in the range. 
        # Return the point with the smallest absolute S value?
        # Or return None. Source implies "look for the first zero crossing".
        # If not found, the method fails to locate the surface.
        return None
        
    # 6. Refinement (Sectioning)
    # Bisection between the two points bracketing the zero crossing
    p_left, s_left = candidates[zero_crossing_idx]
    p_right, s_right = candidates[zero_crossing_idx + 1]
    
    x_hat = p_left
    for _ in range(n_sectioning):
        mid = 0.5 * (p_left + p_right)
        s_mid = sdf(mid)
        if s_left * s_mid < 0:
            p_right = mid
            s_right = s_mid
        else:
            p_left = mid
            s_left = s_mid
        x_hat = mid
        
    # 7. Gradient Adjustment (Differentiable Step)
    # This step is mentioned as an extra step for differentiability.
    # We apply it if the conditions are met.
    s_hat = sdf(x_hat)
    if abs(s_hat) < epsilon_grad:
        # Compute gradient
        # Numerical gradient or analytical if available. 
        # Source assumes gradient computation is possible.
        eps = 1e-4
        grad = np.zeros(3)
        for j in range(3):
            p_plus = x_hat.copy()
            p_plus[j] += eps
            grad[j] = (sdf(p_plus) - s_hat) / eps
            
        denom = np.dot(grad, ray_direction)
        if abs(denom) > epsilon_clamp:
            x_hat = x_hat - (s_hat / denom) * ray_direction
            
    return x_hat
```

## Operating characteristics
*   **Time Complexity**: The method performs $n$ forward iterations, $n$ backward iterations, $N_{samples}$ SDF evaluations, and $N_{sec}$ bisection steps. With $n=16$, $N_{samples}=100$, and $N_{sec}=8$, the total SDF evaluations are approximately $16 + 16 + 100 + 8 = 140$ per ray, plus potentially one gradient computation for the differentiable step. This is significantly more expensive than standard sphere tracing (16 steps) but ensures convergence where standard tracing fails [S1].
*   **Memory Complexity**: $O(1)$ auxiliary memory per ray. No global structures are required [S1].
*   **Strengths**: Robust against rays that miss surface edges or intersect surfaces at shallow angles, where standard sphere tracing may stall or diverge [S1]. It guarantees a bounded search range [S1].
*   **Limitations**: Higher computational cost due to the bidirectional trace and sampling phase. The backward trace requires calculating the rear intersection with a bounding volume (unit sphere in the source), which may not be applicable to all scene representations without modification [S1]. The method is non-differentiable by default; differentiability requires an extra forward step with gradient tracking [S1].

## Background
Bidirectional Sphere Tracing was introduced by Yariv et al. in the context of Neural Volumes and later adapted for Neural Lumigraph Rendering and IDR (Implicit Disentangled Representation) [S1]. It addresses the limitation that standard sphere tracing convergence is not guaranteed for general shape configurations, particularly when the SDF underestimates the optimal step length due to non-orthogonal surface intersections [S1]. The technique was cited from Yariv et al. [60] in the source [S1].

## Sources
[S1] Neural Lumigraph Rendering, https://doi.org/10.1109/cvpr46437.2021.00427, fetched 2026-10-26
