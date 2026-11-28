# Gradient-based Sphere Tracing with Lipschitz Constraints

Also called: 2-Lipschitz Sphere Tracing, Gradient-assisted Sphere Tracing

## Overview
Gradient-based Sphere Tracing with Lipschitz Constraints is an optimization of the sphere tracing algorithm that uses the local gradient of the Signed Distance Function (SDF) to compute larger, safer step sizes, particularly improving performance near silhouettes and backward-facing surfaces [S1]. It sets itself apart from standard sphere tracing by combining the first and second Lipschitz constraints derived from the gradient and curvature, allowing for significantly fewer steps in grazing scenarios where standard methods stall [S1].

## How it works
The method improves upon the standard sphere tracing step size $d = f(p) / L$ by utilizing local derivative information to tighten the bound on the distance to the root, ensuring the step does not overshoot the isosurface [S1].

**Inputs and Outputs:**
- Input: Ray origin $O$, ray direction $D$, SDF function $f(p)$, and its gradient $\nabla f(p)$.
- Output: Intersection distance $t$ along the ray.

**Data Structures:**
- No specific complex data structures are required for the core stepping logic, though the method is often implemented within a Bounding Volume Hierarchy (BVH) for FBM fields to dynamically compute local Lipschitz constants [S1].

**Numbered Steps:**
1.  **Evaluate Field:** At current position $p$, evaluate the SDF value $f(p)$ and its gradient $\nabla f(p)$.
2.  **Compute Standard Step:** Calculate the initial candidate step size using the gradient magnitude:
    $$ d_{grad} = \frac{f(p)}{|\nabla f(p)|} $$
    [G] If $|\nabla f(p)|$ is near zero, fall back to a global Lipschitz constant or a minimum step size to avoid division by zero.
3.  **Apply 2-Lipschitz Constraint:** Refine the step size using the second-order Lipschitz constraint (curvature bound) from [AZ23]. The safe step size $d$ is the minimum of the gradient-based step and the curvature-constrained step:
    $$ d = \min \left( \frac{f(p)}{|\nabla f(p)|}, \sqrt{\frac{2 f(p)}{|\nabla^2 f(p)|}} \right) $$
    [G] The term $|\nabla^2 f(p)|$ represents the magnitude of the Hessian or a bound on the second derivative. In practice, for 2-Lipschitz functions, this constraint ensures the step respects the curvature bounds [S1].
4.  **Advance:** Update position $p \leftarrow p + d \cdot D$.
5.  **Terminate:** If $f(p) < \epsilon$ (threshold), return $d$. Otherwise, repeat from Step 1.

**Parameters:**
- **Lipschitz Constant $L$:** For general FBM, this is dynamically computed using a BVH to be much smaller than the global constant, reducing the step size penalty in empty space [S1].
- **Threshold $\epsilon$:** Standard sphere tracing termination threshold (e.g., $10^{-4}$).

**Edge Cases and Pitfalls:**
- **FBM Slowdowns:** The standard formulation of [AZ23] can cause slowdowns for certain functions like Fractional Brownian Motion (FBM) due to improper handling of cascaded sums [S1]. A modified formulation is required to properly handle the gradient and Lipschitz constants for these nested structures [S1].
- **Grazing Angles:** Standard sphere tracing takes very small steps near silhouettes; this method mitigates this by using the gradient to predict the distance more accurately [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_sphere_trace(p, ray_dir, sdf_func, grad_func, hess_bound_func, epsilon=1e-4):
    """
    Performs sphere tracing using gradient and 2-Lipschitz constraints.
    
    Args:
        p: Current position (vec3)
        ray_dir: Ray direction (vec3, normalized)
        sdf_func: Function returning f(p)
        grad_func: Function returning gradient grad_f(p)
        hess_bound_func: Function returning bound on second derivative |nabla^2 f|
        epsilon: Termination threshold
        
    Returns:
        Intersection distance t
    """
    t = 0.0
    while True:
        f_val = sdf_func(p)
        
        # Termination check
        if f_val < epsilon:
            return t
            
        # Compute gradient
        grad_f = grad_func(p)
        grad_mag = length(grad_f)
        
        # Guard against zero gradient
        if grad_mag < 1e-8:
            step = epsilon # Fallback or global Lipschitz step
        else:
            # Step 1: Gradient-based step
            step_grad = f_val / grad_mag
            
            # Step 2: 2-Lipschitz (curvature) constraint
            # Using bound on second derivative to ensure safety
            hess_bound = hess_bound_func(p)
            
            # Avoid division by zero in curvature term
            if hess_bound < 1e-8:
                step_curv = step_grad # Fall back to gradient step if curvature bound is negligible
            else:
                # Formula derived from 2-Lipschitz constraint: d <= sqrt(2*f / |nabla^2 f|)
                step_curv = sqrt(2.0 * f_val / hess_bound)
            
            # Take the minimum safe step
            step = min(step_grad, step_curv)
            
        # Advance
        p = p + step * ray_dir
        t += step
```

## Subtopics

### [Idempotent Eikonal Ray Marching](3810-idempotent-eikonal-ray-marching.md)
Idempotent Eikonal Ray Marching is a ray-surface intersection method that formulates intersection as a curvature-accelerated contraction operator on the Eikonal distance field to solve the computational stall of sphere tracing near grazing angles and silhouette edges. It sets itself apart from its parent, Gradient-based Sphere Tracing with Lipschitz Constraints, by selectively activating second-order Halley contraction only within grazing silhouette horizons while maintaining single-query Lipschitz advances in open space, thereby truncating the grazing stall tail by up to 4×.

### [Segment Tracing with Local Lipschitz Bounds](4094-segment-tracing-with-local-lipschitz-bounds.md)
Segment Tracing is a ray marching algorithm that computes a local Lipschitz bound $\lambda(e)$ for each ray segment $e$ to determine safe, larger stepping distances, significantly reducing the number of field function queries compared to global bound methods like Sphere Tracing. It solves the problem of inefficient stepping near grazing surfaces and complex implicit structures by adapting the step size to the local geometry's smoothness, allowing for convergence in fewer steps while maintaining safety via an overstepping check. Unlike its siblings, it explicitly bounds the gradient and Jacobian of deformed fields to compute these local constraints.

### [Structure-aware sampling](21266-structure-aware-sampling.md)
Structure-aware sampling is an adaptive ray marching algorithm that uses a Signed Distance Function (SDF) and its filtered gradient to determine step sizes, ensuring efficient traversal and accurate surface intersection. It leverages structural information from LiDAR point clouds and neural fields to skip free space via an occupancy map and avoid false revert steps in unconverged neural distance fields (NDFs) by incorporating a predicted scale parameter. It sets itself apart from standard sphere tracing by using a filtered slope to modulate step size and a specific stability check involving the scale $\beta_i$ to handle the uncertainty of learned SDFs.

## Operating characteristics
- **Time Complexity:** Reduces the number of SDF evaluations significantly compared to standard sphere tracing, especially in empty space and near silhouettes. Speedups of up to 12-fold are reported for opaque or constant semi-transparent scenes with high-complexity FBM [S1].
- **Memory Complexity:** Requires storage for the BVH if used for dynamic Lipschitz constant computation, but the core stepping logic is $O(1)$ auxiliary space [S1].
- **Strengths:** Effectively handles grazing angles and backward-facing surfaces where standard sphere tracing fails due to small step sizes. Compatible with semi-transparency and hypertextures [S1].
- **Limitations:** Requires computation of the gradient and potentially the Hessian bound, which increases per-step cost. The standard [AZ23] formulation may cause slowdowns for FBM without the proposed modifications [S1].
- **When to use:** Ideal for rendering procedural volumetric noise (FBM) and complex implicit surfaces where grazing angles are common and global Lipschitz constants are too conservative [S1].

## Background
The method is based on the work of [AZ23], which introduced the use of 2-Lipschitz constraints for sphere tracing [S1]. The specific application to FBM and the identification of the slowdown issue with cascaded sums are detailed in [S1]. The technique builds on the foundational sphere tracing method by [Har96] and Lipschitz methods by [Mit90] and [KB89] [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
