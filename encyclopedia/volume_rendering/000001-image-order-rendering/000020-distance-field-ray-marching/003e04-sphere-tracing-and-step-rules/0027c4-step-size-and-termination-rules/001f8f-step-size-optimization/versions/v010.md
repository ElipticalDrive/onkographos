# Step Size Optimization
Also called: Adaptive Stepping, Dynamic Step Sizing

## Overview
Step Size Optimization determines the integration distance $\Delta t$ for each iteration of a ray marching loop to balance rendering accuracy against computational cost. It solves the problem of fixed-step inefficiency, where uniform sampling wastes samples in empty space or misses high-frequency details in complex regions. This family is distinguished from [Sphere Tracing](1674-sphere-tracing.md) by relaxing the strict Lipschitz safety constraint ($\Delta t \le f(\mathbf{p})$) in favor of heuristic, error-based, or geometric bounds that allow larger steps in safe regions, and from [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by dynamically varying $\Delta t$ per sample rather than using a global constant.

## Shared principles
All methods in this family operate on a ray $r(t) = \mathbf{o} + t\mathbf{d}$, where $\mathbf{o}$ is the origin and $\mathbf{d}$ is the normalized direction vector. The core loop iteratively updates the accumulated distance $t$ and evaluates a scene function $f(\mathbf{p})$ at the current position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.

**Step Size Determination:**
The step size $\Delta t_i$ at iteration $i$ is constrained by the local geometry to ensure safety or efficiency.
1.  **Safety Constraint:** For Signed Distance Functions (SDFs) satisfying the Lipschitz condition with constant $\lambda=1$, the step size must satisfy $\Delta t \le f(\mathbf{p})$ to guarantee the ray does not penetrate or overshoot the surface [Parent]. If the function has a Lipschitz constant $\lambda > 1$, the safe step size is scaled: $\Delta t = \frac{|f(\mathbf{p})|}{\lambda}$ [S2, S2054].
2.  **Adaptation:** In optimized or volumetric contexts, $\Delta t$ may be dynamically adjusted based on accumulated density, distance from the camera, or low-resolution approximations to reduce sample counts in less critical areas [S1, S2054].

**Termination Conditions:**
The loop terminates when one or more of the following conditions are met:
1.  **Convergence (Hit):** The ray has reached the surface or sufficient density. For SDFs, this is typically $f(\mathbf{p}_i) < \epsilon$ [Parent]. For volumes, this may be based on accumulated opacity or density thresholds.
2.  **Escape (Miss):** The ray has exited the scene or bounding volume, typically when accumulated distance $t > t_{max}$ [Parent].
3.  **Resource Limit:** A maximum number of iterations $N_{max}$ is reached to prevent infinite loops [Parent].
4.  **Stagnation/Oscillation:** The step size or distance value fails to decrease significantly, indicating the ray is stuck or oscillating near a surface or in noise [Parent].

## Subtopics

### [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md)
Safe Raymarching Step Size is a method for determining the distance to advance along a ray in Signed Distance Field (SDF) rendering to guarantee that the ray never overshoots or penetrates the implicit surface. It solves the problem of geometric artifacts and missed intersections caused by inaccurate distance estimates, particularly when SDFs are distorted by boolean operations or domain repetition. Unlike standard sphere tracing which assumes the SDF is a perfect lower bound, this method introduces safety margins (clamping, ratios) or grid-aware bounds to ensure numerical stability. It sets itself apart from siblings like [Adaptive Step Size Control](9843-adaptive-step-size-control.md) by prioritizing strict non-penetration guarantees over aggressive convergence speed, often at the cost of increased sample counts.

### [Compute Minimal Sample Distance Per Pixel](2684-compute-minimal-sample-distance-per-pixel.md)
This method computes the minimal sample distance per pixel for volume rendering by projecting each dataset cell axis onto the viewport, accounting for camera and volume transforms. It serves as a geometric heuristic to determine the appropriate step size for ray marching, ensuring that samples are dense enough to resolve the smallest visible voxel features. It differs from [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by relying on explicit axis projection rather than conservative bounds, and from [Error Estimate](2768-error-estimate.md) by using geometric projection instead of field value derivatives.

### [Error Estimate](2768-error-estimate.md)
Error Estimate calculates a relative error metric to guide adaptive stepping in distance-field ray marching. It solves the problem of determining an appropriate step size by evaluating the local variation of the scalar field relative to its magnitude, ensuring that steps are small enough to capture detail where the field changes rapidly or is small, but large enough to maintain performance where the field is stable. It sets itself apart from siblings like [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by using a relative error bound based on the field value itself rather than a fixed geometric bound or derivative-based Lipschitz constant.

### [Gap-Limited Ray Marching](4412-gap-limited-ray-marching.md)
Gap-Limited Ray Marching is a step-size optimization for heightfield terrain rendering that prevents the ray from overshooting surface features, such as ridgelines, which can cause visual holes. Unlike standard sphere tracing which relies on a global distance field, this method uses a step size proportional to the distance traveled, bounded by half the vertical gap to the surface. It builds on [Step Size and Termination Rules](4132-step-size-and-termination-rules.md) by introducing a geometric constraint specific to heightfields.

### [Adaptive Volume Rendering Integration](9817-adaptive-volume-rendering-integration.md)
Adaptive Volume Rendering Integration is an iterative numerical method for evaluating the Volume Rendering Integral (VRI) that uses adaptive step sizes for both the inner (transparency) and outer (color) integrals to control accuracy and efficiency. It sets itself apart from recursive adaptive methods by using an iterative approach that avoids recursion depth issues on GPUs and allows re-initialization of step sizes, enabling coherent control of inner and outer integration steps without excessive recomputation.

### [Adaptive Step Size Control](9843-adaptive-step-size-control.md)
Adaptive Step Size Control is a family of algorithms in distance-field ray marching that dynamically adjusts the integration step length $\Delta t$ along a ray based on local scene properties (such as density, gradient magnitude, or empty space) to optimize the trade-off between rendering quality and computational cost. It distinguishes itself from fixed-step methods by skipping redundant samples in empty regions and refining samples in high-frequency areas, and from sibling methods like [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by often employing heuristic or error-driven feedback loops rather than purely geometric safety bounds.

### Convexity Enhancement
Convexity Enhancement leverages the geometric property of convex sets to increase the step size in sphere tracing, achieving faster convergence than standard methods [S1]. For a convex set defined by an SDF, the line segment from the current point to the intersection of the ray with the tangent plane at that point does not intersect the surface except possibly at the endpoint [S1]. This allows the ray to "jump" closer to the surface than the standard SDF value would permit, effectively approximating Newton's method and providing quadratic convergence [S1].

The step is calculated by finding the intersection of the ray with the tangent plane. Given a point $\mathbf{p}$ and surface normal $\mathbf{n}(\mathbf{p}) = \nabla f(\mathbf{p})$, the tangent plane intersection is found at distance $d(\mathbf{p}) / \|\mathbf{n}(\mathbf{p})\|$ along the normal direction, projected onto the ray [S1]. The update step is:
$$ \mathbf{p}_{new} = \mathbf{p} + \frac{f(\mathbf{p})}{\mathbf{n}(\mathbf{p}) \cdot \mathbf{d}} \mathbf{d} $$
If $f(\mathbf{p}) < 0$, the ray is inside the convex object and does not intersect the surface from this direction [S1].

This optimization significantly reduces execution time; in tests, it reduced rendering time for a single sphere by 31% (from 2:00 to 1:23) and for the Zeno logo by 27% (from 26:29 to 19:23) [S1]. A critical pitfall is handling rays parallel to horizon lines (e.g., a ground plane); standard sphere tracing would step indefinitely, but Corollary 4.1 allows early termination if the ray has stepped beyond the object's bounds [S1].

### FBM Performance Optimization
Fractal Brownian Motion (FBM) optimization reduces the computational cost of evaluating noise-based SDFs or density fields by adapting the complexity of the noise function to the viewing conditions [S2]. The primary technique is reducing the octave count based on the distance $t$ from the camera or the time parameter, using logarithmic scaling: $\text{oct} = \lfloor \log_2(1.0 + t \cdot 0.5) \rfloor$ [S2]. This ensures distant objects use fewer octaves, lowering sample count and arithmetic intensity.

Additional optimizations include:
*   **Multi-level LOD:** Assigning different octave counts (e.g., 3, 9, 16) to different resolution tiers of the scene [S2].
*   **Texture Sampling:** Replacing mathematical hash functions for noise generation with texture lookups, which are often faster on GPU hardware [S2].
*   **Directional Derivatives:** Computing normals using a single directional derivative instead of a full 3-component gradient, reducing the number of noise evaluations from 3 to 1 [S2].
*   **Early Termination:** Breaking the FBM summation loop if the accumulated amplitude exceeds a threshold (e.g., 0.99), as further octaves contribute negligibly to the final value [S2].

These techniques are often combined with adaptive step sizes, where the step length $\Delta t$ is scaled by distance (e.g., $\Delta t = \max(0.05, 0.02 \cdot t)$) to match the decreasing frequency of distant features [S2].

## Background
The concept of optimizing step size in ray marching originated with the introduction of Sphere Tracing by John Williams et al. in 1996 [S1]. While the original method used a fixed step size equal to the SDF value, subsequent research identified inefficiencies in this approach, particularly for convex objects and complex noise fields. The "Convexity Enhancement" was introduced in the same foundational paper to accelerate convergence by leveraging tangent plane intersections [S1]. Later developments in volume rendering and procedural noise (FBM) introduced heuristic and error-based adaptive stepping to handle the high computational cost of evaluating complex scalar fields [S2]. Modern implementations often combine these geometric safety bounds with error estimates and LOD techniques to achieve real-time performance [S1, S2].

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray Tracing of Implicit Surfaces, https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
