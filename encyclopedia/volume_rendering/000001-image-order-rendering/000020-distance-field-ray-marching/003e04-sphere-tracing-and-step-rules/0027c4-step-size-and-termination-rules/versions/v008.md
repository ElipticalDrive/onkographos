# Step Size and Termination Rules

Also called: Ray Marching Step Control, Marching Convergence

## Overview
This family defines the logic for determining the distance $\Delta t$ to advance a ray along its direction $\mathbf{d}$ and the conditions under which the marching loop terminates. It solves the problem of balancing rendering accuracy against computational cost, ensuring that rays do not overshoot geometry, get trapped in infinite loops, or waste samples in empty space. What sets this family apart from its siblings is its focus on the *step size calculation* and *exit criteria* themselves, rather than the acceleration structures (like BVHs) or the specific SDF composition methods. It builds on [Sphere Tracing and Step Rules](9643-sphere-tracing-and-step-rules.md).

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

### [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md)
Standard Constant Step Ray Marching is an iterative root-finding technique for determining the intersection of a ray with an implicit surface or evaluating volumetric fields by stepping along the ray at fixed intervals. It solves the problem of rendering implicit surfaces defined by Signed Distance Functions (SDFs) or scalar fields without requiring analytical intersection solutions or polygonal mesh rasterization. Unlike [Sphere Tracing](1674-sphere-tracing.md), which dynamically adjusts step size based on the local SDF value to guarantee no overshooting, this method uses a predefined constant step size $\Delta t$, simplifying implementation but requiring careful tuning to avoid aliasing or missed details. It differs from [Step Size Optimization](2054-step-size-optimization.md) by ignoring local geometry constraints for performance, and from [SDF Ray Marching with Grid Lookup](5017-sdf-ray-marching-with-grid-lookup.md) by not using interpolation for precise intersection detection.

### [Sphere Tracing](1674-sphere-tracing.md)
Sphere Tracing is an iterative ray-marching algorithm for rendering implicit surfaces defined by Signed Distance Functions (SDFs). It solves the problem of finding the intersection between a camera ray and a complex, procedurally defined geometry without explicit polygonal meshes. What sets it apart from its siblings, such as [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md), is its use of the SDF value at the current position as a guaranteed lower bound on the distance to the nearest surface, allowing for adaptive, non-uniform step sizes that ensure the ray never overshoots the geometry. Unlike [Interval Raymarching](13020-interval-raymarching.md) which handles uncertainty, or [Cone Intersection](1553-cone-intersection.md) which handles antialiasing via cone volumes, standard Sphere Tracing relies on the strict Lipschitz continuity of the SDF to guarantee convergence.

### [Step Size Optimization](2054-step-size-optimization.md)
Step Size Optimization determines the integration distance $\Delta t$ for each iteration of a ray marching loop to balance rendering accuracy against computational cost. It solves the problem of fixed-step inefficiency, where uniform sampling wastes samples in empty space or misses high-frequency details in complex regions. This family is distinguished from [Sphere Tracing](1674-sphere-tracing.md) by relaxing the strict Lipschitz safety constraint ($\Delta t \le f(\mathbf{p})$) in favor of heuristic, error-based, or geometric bounds that allow larger steps in safe regions, and from [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by dynamically varying $\Delta t$ per sample rather than using a global constant.

### [Sphere Tracing Termination and Clipping](2635-sphere-tracing-termination-and-clipping.md)
[G] Sphere Tracing Termination and Clipping defines the conditions under which the iterative ray-marching loop halts, ensuring the algorithm converges to a surface intersection or correctly identifies a miss. It solves the problem of infinite loops and geometric inaccuracies by combining convergence thresholds, oscillation detection, and spatial clipping. It distinguishes itself from sibling methods like [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by strictly enforcing the Lipschitz safety constraint ($\Delta t \le f(\mathbf{p})$) for termination logic, and from [Sphere Tracing](1674-sphere-tracing.md) by explicitly handling edge cases such as oscillation and far-plane clipping to guarantee robustness in real-time rendering.

### [SDF Scaling and Early Ray Termination for Fast Sphere Tracing](3807-sdf-scaling-and-early-ray-termination-for-fast-sphere-tracin.md)
This method accelerates sphere tracing by using a two-pass approach: a low-resolution pre-pass that applies Signed Distance Function (SDF) scaling and early termination to generate a coarse distance buffer, followed by a full-resolution pass that uses this buffer to initialize ray origins closer to surfaces. It distinguishes itself from sibling methods like [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) and [Sphere Tracing](1674-sphere-tracing.md) by explicitly modifying the SDF geometry in the pre-pass to prevent missed small features and reduce step counts, while avoiding the memory overhead of pyramidal buffers used in [Inverse Pyramid Rendering](1080-standard-constant-step-ray-marching.md).

### [Interval Raymarching](13020-interval-raymarching.md)
Interval Raymarching is a technique for accelerating Signed Distance Field (SDF) raymarching by evaluating the SDF over an interval of the ray using affine or interval arithmetic, rather than at a single point. It solves the problem of slow convergence in standard sphere tracing by enabling reliable linear interpolation (secant method) to find surface intersections, aiming for the $\phi$ degree of convergence (golden ratio) instead of the linear convergence of sphere tracing. Unlike sphere tracing, which guarantees safety via Lipschitz constraints, Interval Raymarching relies on conservative-estimate SDFs to ensure correctness when leaping across potential intersections.

#### Adaptive Minimum Step Size

### definition
- [S1, unofficial] Dynamic adjustment of the minimum step size based on distance from camera and accumulated density to reduce samples in less important areas.

### step
- [S1, unofficial] Inside the cloud (distanceField <= 0), increase minStepSize by STEP_INSCREASE_OVER_DENSITY multiplied by the current sampleDensity.
- [S1, unofficial] After advancing the ray, increase minStepSize by STEP_INCREASE_OVER_DISTANCE multiplied by the stepSize used.

### parameter
- [S1, unofficial] STEP_INSCREASE_OVER_DENSITY: Constant controlling how much minStepSize increases per unit of sampleDensity.
- [S1, unofficial] STEP_INCREASE_OVER_DISTANCE: Constant controlling how much minStepSize increases per unit of stepSize (distance traveled).

### comparison
- [S1, unofficial] Increases step size further from camera (less detail needed) and behind dense areas (less light contribution), unlike fixed step sizes.

#### Cone Step Mapping

### definition
- [S2, unofficial] Cone step mapping and relaxed cone step mapping are space leaping acceleration techniques used in ray marching.

#### Early Ray Marching Termination (Threshold)

### definition
- [S3, unofficial] An optimization to stop ray marching when the accumulated transmission falls below a small threshold, saving computation in dense volumes.

### parameter
- [S3, unofficial] Threshold: A small value (e.g., $10^{-3}$ or 0.001) representing the minimum transmission. Below this, the contribution to the final color is considered negligible.

### pitfall
- [S3, unofficial] Simply stopping and ignoring the remaining volume contribution is statistically incorrect and introduces bias. It is acceptable for visual rendering but not for high-precision scientific simulation (e.g., thermonuclear engineering).

## Background
The concept of sphere tracing was introduced by John Hart in 1996 [S2, unofficial]. It is widely used in real-time rendering of implicit surfaces, procedural geometry, and volumetric effects such as clouds. The specific optimizations like adaptive step sizes for clouds and early termination thresholds are common in shader programming for games and interactive applications [S1, unofficial][S3, unofficial].

## Sources
[S1] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S2] What is Ray Marching? Is Sphere Tracing the same thing?, https://computergraphics.stackexchange.com/questions/161, fetched 2026-10-29 (unofficial)
[S3] "Volume Rendering for Developers" Group Read Part 1 [2023-03-30], https://www.youtube.com/watch?v=mZzMVD4LQvk, fetched 2026-10-26 (unofficial)
