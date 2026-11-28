# Step Size Optimization

Also called: Adaptive Step Sizing, Dynamic Step Control

## Overview
Step Size Optimization encompasses techniques that dynamically determine the distance $\Delta t$ to advance a ray during volume rendering or implicit surface marching, replacing fixed-step methods to balance computational cost and visual fidelity [G]. It solves the inefficiency of uniform sampling in empty space and the aliasing/penetration artifacts of steps that are too large for local geometry [G]. Unlike [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md), which uses a uniform $\Delta t$, this family adapts the step length based on local scene properties such as Signed Distance Field (SDF) values, density gradients, or geometric gaps. It differs from [Sphere Tracing](1674-sphere-tracing.md) in that while sphere tracing strictly uses the SDF value as a lower bound for the step (often with a relaxation factor), step size optimization includes broader strategies like error estimation, gap limiting, and integration-based adaptive stepping that may not rely solely on the geometric distance bound.

## Shared principles
All methods in this family operate on a ray defined by origin $\mathbf{o}$ and normalized direction $\mathbf{d}$, parameterized by distance $t$ [G]. The core loop evaluates a scene function (SDF, density field, or volume texture) at the current position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ to determine the next step size $\Delta t$ and check for termination [G].

**Termination Conditions:**
1.  **Surface Hit:** The ray has intersected the target geometry [G]. For SDFs, this is typically $|f(\mathbf{p})| < \epsilon$ or $f(\mathbf{p}) \le 0$ [G]. For volumes, this may be a density threshold or opacity accumulation [G].
2.  **Escape/Miss:** The ray has traveled beyond a maximum distance $t_{max}$ or exceeded a maximum iteration count $N_{max}$ [G].
3.  **Early Termination (Volumes):** Accumulated opacity $\alpha_{acc}$ exceeds a threshold (e.g., $1 - \epsilon$), or accumulated transmission falls below a minimum threshold, allowing the ray to stop early as further contributions are negligible [G].

**Step Size Determination:**
The step size $\Delta t$ is computed based on the local scene properties:
*   **SDF-based:** $\Delta t$ is often derived from the SDF value $f(\mathbf{p})$, potentially scaled by a relaxation factor or Lipschitz constant $\lambda$ [G].
*   **Volume-based:** $\Delta t$ may be fixed, adaptive based on density gradients, or constrained by minimum/maximum bounds to prevent aliasing or excessive computation [G].
*   **Safety:** Steps must be chosen to ensure the ray does not penetrate the surface or skip over significant volumetric features, often requiring conservative clamping [G].

**Pseudocode Structure:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_loop(ray_origin, ray_direction, scene_func, max_dist, max_steps, epsilon, min_step=1e-4, max_step=1.0):
    """
    Generic adaptive ray marching loop implementing shared principles.
    scene_func(p) returns (val, alpha) where val is SDF/density and alpha is opacity.
    """
    t = 0.0
    alpha_acc = 0.0
    
    for i in range(max_steps):
        # Escape/Miss check
        if t >= max_dist:
            return False, t, alpha_acc
        
        # Evaluate scene
        p = ray_origin + ray_direction * t
        val, alpha = scene_func(p)
        
        # Surface Hit check (SDF)
        if abs(val) < epsilon:
            return True, t, alpha_acc
            
        # Early Exit check (Volume)
        if alpha_acc >= 1.0 - epsilon:
            return False, t, alpha_acc
            
        # Adaptive Step Calculation
        # Use SDF/density magnitude as base step, fallback to max_step if negative/zero
        step = abs(val) if val > 0 else max_step
        step = max(min_step, min(step, max_step))
        
        # Guard against zero/negative steps to prevent infinite loops
        if step <= 0:
            step = min_step
            
        t += step
        alpha_acc += alpha * (1.0 - alpha_acc)
        
    return False, t, alpha_acc
```

## Subtopics

### [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md)
Uses the local SDF value as an upper bound for the step size, scaled by a ratio, to prevent surface penetration. Chosen when geometric safety is paramount and a valid SDF is available.

### [Compute Minimal Sample Distance Per Pixel](2684-compute-minimal-sample-distance-per-pixel.md)
Calculates the minimal sample distance based on the projected voxel thickness along the view normal using L1 or L2 norms. Chosen for volume rendering to ensure consistent voxel coverage and prevent gaps or overlaps in oblique views.

### [Error Estimate](2768-error-estimate.md)
Determines step size based on a relative error metric derived from the field's derivative and magnitude. Chosen when high precision is required in regions of high field variation, avoiding small steps in high-magnitude regions.

### [Gap-Limited Ray Marching](4412-gap-limited-ray-marching.md)
Bounds the step size by half the vertical gap to the surface for heightfields. Chosen for terrain rendering to prevent stalling and overshooting on non-distance fields.

### [Adaptive Volume Rendering Integration](9817-adaptive-volume-rendering-integration.md)
Iteratively adjusts step sizes for the volume rendering integral using Simpson's rule error estimation. Chosen for high-fidelity volume rendering where accuracy of the integrated color/opacity is critical.

### [Adaptive Step Size Control](9843-adaptive-step-size-control.md)
Dynamically adjusts step size based on local scene metrics (density, variance) to balance cost and fidelity. Chosen as a general-purpose adaptive strategy for complex volumetric data.

### Convexity Enhancement
Convexity Enhancement leverages the geometric property of convex sets to increase step size and accelerate convergence in sphere tracing [S2]. For a convex set $A$ defined by a signed distance function $d(\mathbf{p})$, the line segment from the current point $\mathbf{p}$ to the intersection of the ray with the tangent plane at $\mathbf{p}$ does not intersect the surface except possibly at the endpoint [S2]. The step is calculated as:
$$ \mathbf{p}_{next} = \text{lerp}(\mathbf{p}, \mathbf{p} + d(\mathbf{p}) \frac{\mathbf{n}(\mathbf{p})}{\|\mathbf{n}(\mathbf{p})\|}) = \mathbf{p} + d(\mathbf{p}) \mathbf{v} $$
where $\mathbf{n}(\mathbf{p})$ is the gradient (normal) and $\mathbf{v}$ is the normalized direction towards the tangent plane intersection [S2]. If $d(\mathbf{p}) < 0$, the ray has passed the object and does not intersect it, allowing for early termination (Corollary 4.1) [S2]. This method causes sphere tracing to converge quadratically, similar to Newton's method, rather than linearly [S2]. In benchmarks, it reduced execution time for a single sphere from 100% to 69% and for the Zeno logo from 100% to 73% [S2]. A pitfall is that rays parallel to a horizon line (e.g., a ground plane) would otherwise step indefinitely; Corollary 4.1 avoids this by detecting when the ray has stepped beyond the object [S2].

### FBM Performance Optimization
Fractal Brownian Motion (FBM) performance optimization involves reducing the computational cost of noise evaluation during ray marching . Key techniques include:
1.  **Reducing Octave Count:** Use fewer octaves for distant objects based on distance $t$ or time, e.g., `int oct = 5 - int(log2(1.0 + t * 0.5))` .
2.  **Multi-level LOD:** Assign different octave counts to different resolutions (e.g., 3, 9, 16 octaves) .
3.  **Texture Sampling:** Use texture sampling instead of mathematical hash functions for noise generation to leverage hardware acceleration .
4.  **Loop Unrolling:** Manually unroll small loops and slightly vary lacunarity to improve instruction pipelining .
5.  **Adaptive Step Size:** Use distance-based adaptive stepping, e.g., `float dt = max(0.05, 0.02 * t)` .
6.  **Directional Derivative:** Compute normals using a directional derivative instead of a full gradient, reducing sample count from 3 to 1 .
7.  **Early Termination:** Break the FBM summation loop if the accumulated amplitude exceeds a threshold (e.g., 0.99) .

## Background
Step size optimization evolved from the limitations of fixed-step ray marching, which either wasted samples in empty space or missed details with large steps [G]. The specific use of L1 norms for volume slice stepping was identified to fix overlap artifacts in oblique views, leading to implementation requirements in libraries like Cornerstone3D [S1]. Convexity enhancements were introduced to accelerate convergence for convex primitives, providing quadratic convergence rates [S2]. FBM optimizations address the high cost of procedural noise in ray marching, particularly for terrain and ocean rendering . Adaptive integration methods for volume rendering further refined step control by treating the rendering equation as a numerical integration problem [S1].

## Sources
[S1] A volume viewport steps an oblique slice by the L2 spacing, and shows the same voxels on two slices, https://github.com/cornerstonejs/cornerstone3D/issues/2912, fetched 2026-10-27
[S2] (PDF) Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
