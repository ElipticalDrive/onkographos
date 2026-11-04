# Gradient-based Sphere Tracing with Lipschitz Constraints

Also called: 2-Lipschitz Sphere Tracing, Gradient-assisted Sphere Tracing

## Overview
Gradient-based Sphere Tracing with Lipschitz Constraints is an optimization for ray marching that uses the local gradient of the Signed Distance Function (SDF) to compute larger, safer step sizes than standard sphere tracing, particularly near silhouettes and backward-facing surfaces [S1]. It sets itself apart from standard sphere tracing by combining the first and second Lipschitz constraints to account for curvature, and from its siblings like [Adaptive Lipschitz Constant (ALC)](1883-adaptive-lipschitz-constant-alc.md) by explicitly leveraging gradient magnitude rather than just bounding volume hierarchies or global constants [S1].

## How it works
**Inputs:** Ray origin $O$, ray direction $D$, SDF function $f(p)$, and its gradient $\nabla f(p)$ [S1].
**Outputs:** Intersection distance $t$ along the ray [S1].

**Data Structures:**
- `p`: Current 3D position (vec3) [G].
- `grad_f`: Gradient vector $\nabla f(p)$ (vec3) [S1].
- `L`: Local Lipschitz constant bound for the gradient magnitude (scalar) [S1].
- `L2`: Local Lipschitz constant bound for the Hessian/curvature (scalar) [S1].

**Steps:**
1. Initialize `p = O` [G].
2. Evaluate $f(p)$ and $\nabla f(p)$ [S1].
3. Calculate the standard sphere tracing step size bound using the gradient magnitude:
   $$ s_1 = \frac{f(p)}{|\nabla f(p)|} $$
   This step assumes the function is 1-Lipschitz with constant $|\nabla f(p)|$ [S1].
4. Apply the 2-Lipschitz constraint to refine the step size based on curvature bounds [S1]. The step size $s$ must satisfy the condition that the function does not change faster than allowed by its second derivative [S1]. The refined step size $s_{final}$ is derived from the quadratic bound imposed by the 2-Lipschitz property:
   $$ s_{final} = \min \left( s_1, \frac{-|\nabla f(p)| + \sqrt{|\nabla f(p)|^2 + 2 L_2 f(p)}}{L_2} \right) $$
   Where $L_2$ is the bound on the Hessian norm (or the Lipschitz constant of the gradient) [S1]. Note: If $f(p) \le 0$, the ray has hit or passed the surface; terminate [S1].
5. Advance the ray: `p = p + s_final * D` [S1].
6. Repeat from step 2 until convergence or max steps [S1].

**Parameters:**
- `L` (Lipschitz constant of $f$): For FBM, this is dynamically computed using a Bounding Volume Hierarchy (BVH) to be much smaller than the global constant [S1].
- `L2` (Lipschitz constant of $\nabla f$): Bounding the curvature. For FBM, this requires careful handling of cascaded sums [S1].

**Pitfalls:**
- Standard formulations of the 2-Lipschitz constraint (e.g., [AZ23]) can cause slowdowns for certain functions like FBM if the gradient and Lipschitz constants for cascaded sums are not handled properly [S1]. A specific formulation is required to fix this issue [S1].
- Precision issues may arise when $f(p)$ is very small near the surface [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gradient_based_sphere_trace(O, D, f, grad_f, L2_bound, max_steps=100):
    p = O
    for _ in range(max_steps):
        val = f(p)
        if val <= 0.0:
            return p
        grad = grad_f(p)
        grad_norm = length(grad)
        if grad_norm < 1e-6:
            step = 1e-4
        else:
            step1 = val / grad_norm
            if L2_bound > 1e-6:
                discriminant = grad_norm * grad_norm + 2 * L2_bound * val
                if discriminant < 0:
                    discriminant = 0
                step2 = (-grad_norm + sqrt(discriminant)) / L2_bound
                step = min(step1, step2)
            else:
                step = step1
        p = p + step * D
    return p
```

## Subtopics

### [Idempotent Eikonal Ray Marching](3810-idempotent-eikonal-ray-marching.md)
Idempotent Eikonal Ray Marching is a ray-surface intersection method that formulates intersection as a curvature-accelerated contraction operator on the Eikonal distance field to solve the computational stall of sphere tracing near grazing angles and silhouette edges. It sets itself apart from its parent, Gradient-based Sphere Tracing with Lipschitz Constraints, by selectively activating second-order Halley contraction only within grazing silhouette horizons while maintaining single-query Lipschitz advances in open space, thereby truncating the grazing stall tail by up to 4×.

### [Segment Tracing with Local Lipschitz Bounds](4094-segment-tracing-with-local-lipschitz-bounds.md)
Segment Tracing is a ray marching algorithm for implicit surfaces that computes a local Lipschitz bound $\lambda(e)$ for each ray segment $e$ to determine safe, adaptive step sizes, significantly reducing field function queries compared to global bound methods. Unlike [Gradient-based Sphere Tracing with Lipschitz Constraints](1936-gradient-based-sphere-tracing-with-lipschitz-constraints.md) which uses gradient magnitude to estimate step size, Segment Tracing rigorously bounds the gradient of the deformed field and the Jacobian of the deformation map to guarantee no surface intersections are skipped, even under complex non-linear warping.

## Operating characteristics
- **Time Complexity:** Reduces the number of SDF evaluations per ray compared to standard sphere tracing, especially near silhouettes [S1].
- **Memory Complexity:** Requires storage for the BVH if used for dynamic Lipschitz constants [S1].
- **Measured Performance:** On opaque or constant semi-transparent scenes, speedups up to 12-fold compared to classical sphere tracing are achieved when combined with BVH-based FBM acceleration [S1]. On non-constant density volumetric scenes, up to 2-fold speedup is observed [S1].
- **Strengths:** Effective near silhouettes and grazing angles where standard sphere tracing takes very small steps [S1]. Compatible with semi-transparency and hypertextures [S1].
- **Limitations:** Requires gradient computation, which may be expensive for some procedural functions. Standard 2-Lipschitz formulations can slow down FBM if not properly implemented [S1].
- **When to use:** When rendering high-frequency procedural noise (FBM) or complex implicit surfaces where grazing angles are common [S1].
- **When not to use:** If gradient computation is prohibitively expensive or if the function is not differentiable everywhere (e.g., Perlin noise with `abs()` requires special handling) [S1].

## Background
Originating from the work on accelerating sphere tracing for procedural volumetric noise, this method builds on the 2-Lipschitz sphere tracing formulation by Aubry et al. [AZ23] [S1]. The specific improvements for FBM cascaded sums and the corrected formulation for gradient-based steps were proposed in "Fast sphere tracing of procedural volumetric noise for very large and detailed scenes" [S1]. It is used in real-time rendering of complex volumetric fields and animated content [S1].

## Sources
[S1] Fast sphere tracing of procedural volumetric noise for very large and detailed scenes, https://doi.org/10.1111/cgf.70072, fetched 2026-10-26
