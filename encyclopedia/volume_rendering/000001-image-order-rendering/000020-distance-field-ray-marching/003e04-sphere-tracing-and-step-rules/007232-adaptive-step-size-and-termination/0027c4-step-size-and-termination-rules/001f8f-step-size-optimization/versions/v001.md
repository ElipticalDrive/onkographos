# Step Size Optimization

Also called: Adaptive Step Size, Dynamic Sampling, Safe Raymarching

## Overview
Step Size Optimization encompasses techniques that vary the distance a ray travels between samples in volume rendering or distance-field ray marching, rather than using a fixed interval. These methods solve the trade-off between rendering quality (avoiding aliasing, overshooting, or missing thin features) and computational cost (reducing the number of expensive density or SDF evaluations). They set themselves apart from [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by dynamically adjusting $\Delta s$ based on local geometry, density, or global bounds, and from [Sphere Tracing](1674-sphere-tracing.md) by often decoupling the step size from the strict Lipschitz constraint of the SDF, or by applying specific heuristics (like gap limits or mip-level doubling) that sphere tracing does not inherently provide.

## Shared principles
All methods in this family share the core mechanism of replacing the constant step $\Delta s = \text{const}$ with a function $\Delta s = f(\mathbf{p}, \mathbf{d}, \text{context})$, where $\mathbf{p}$ is the current ray position, $\mathbf{d}$ is the ray direction, and context includes previous samples, camera state, or dataset metadata.

The general update rule for a ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized) is:
$$ \mathbf{p}_{i+1} = \mathbf{p}_i + \Delta s_i \mathbf{d} $$
$$ t_{i+1} = t_i + \Delta s_i $$

Key shared constraints and goals include:
1.  **Safety/Correctness:** The step size must not cause the ray to "overshoot" or "penetrate" geometry or density features that should have been sampled. For SDFs, this often requires $\Delta s \le d(\mathbf{p})$ (Sphere Tracing) or a conservative fraction thereof. For volumes, it requires sufficient sampling density to satisfy the Nyquist criterion or to capture density gradients accurately.
2.  **Efficiency:** The step size should increase in empty or low-gradient regions to reduce sample count.
3.  **Termination:** Step size changes interact with termination rules (e.g., Early Ray Termination). A larger step size might skip the exact intersection point, requiring post-processing refinement (e.g., binary search).

## Subtopics

### [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md)
Safe Raymarching Step Size guarantees the ray never overshoots the implicit surface even when the SDF is distorted (e.g., by boolean operations or domain repetition). It differs from standard sphere tracing by explicitly clamping SDF values and applying a conservative ratio ($>1.0$) to the step increment, and from [Adaptive Step Size Ray Marching](2312-adaptive-step-size-ray-marching.md) by relying on fixed global bounds rather than local gradient estimation. Choose this when rendering complex CSG models or distorted SDFs where standard sphere tracing produces artifacts.

### [Adaptive Step Size Ray Marching](2312-adaptive-step-size-ray-marching.md)
Adaptive Step Size Ray Marching is a GPU-based direct volume rendering technique that varies step size based on local density and uses temporal accumulation to manage noise. It differs from [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by using a noise buffer to drive step size variation and temporal reprojection, and from [Gap-Limited Ray Marching](4412-gap-limited-ray-marching.md) by its continuous adaptive scaling rather than discrete gap skipping. Choose this for dynamic volumetric data (like clouds or smoke) where temporal coherence can be exploited.

### [Dynamic Sampling Step Adjustment](2442-dynamic-sampling-step-adjustment.md)
Dynamic Sampling Step Adjustment accelerates ray marching through empty regions by exponentially increasing the step size (doubling) and mip level, reducing samples in sparse volumes. It differs from [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) and [Adaptive Step Size Ray Marching](2312-adaptive-step-size-ray-marching.md) by using a discrete doubling strategy rather than continuous estimation, and from [Gap-Limited Ray Marching](4412-gap-limited-ray-marching.md) by not explicitly calculating geometric gaps. Choose this for large, sparse volumes with multi-resolution textures.

### [Compute Minimal Sample Distance Per Pixel](2684-compute-minimal-sample-distance-per-pixel.md)
This method computes a static, geometry-driven minimum step size per pixel by projecting the dataset's cell axes onto the viewport. It ensures sampling density respects voxel resolution. Unlike adaptive methods, it does not adjust during marching but provides a heuristic bound. Choose this when strict adherence to voxel resolution is required to prevent aliasing, regardless of local density.

### [Gap-Limited Ray Marching](4412-gap-limited-ray-marching.md)
Gap-Limited Ray Marching prevents overshooting on heightfield terrain by bounding the step size by half the vertical gap to the surface. Unlike standard distance-field marching which relies on a global Lipschitz constant, this method uses a step size proportional to the distance traveled, bounded by the gap. Choose this for terrain rendering or heightfield-based volumes where ridgelines cause overshooting in standard methods.

### Triangle Inequality Enhancement for Sphere Tracing
An optimization that uses the triangle inequality to bound the distance to objects in a collection, eliminating unnecessary distance computations. It is particularly effective for large assortments of objects. If all primitives are nearly equidistant from the eye, the reduction is minimal until the ray has traversed much of the distance. For 9 spheres/plane, it reduced execution time from 2:53 to 1:19 (46% relative time). For the zeno logo, it reduced time from 26:29 to 19:23 (73% relative time) [S1].

### Convexity Enhancement
Uses knowledge of convexity to increase step size. For a convex set $A$ defined by SDF $d$, the line segment $\text{lerp}(\mathbf{p}, \mathbf{p} + d(\mathbf{p}) \frac{\mathbf{n}(\mathbf{p})}{\|\mathbf{n}(\mathbf{p})\|})$ does not intersect $A$ except possibly at the endpoint. If $d(\mathbf{p}) < 0$, the ray does not intersect. This causes quadratic convergence, similar to Newton's method. It avoids fixed intervals for rays parallel to horizon lines by detecting if the ray has stepped beyond the object. For a single sphere, it reduced time from 2:00 to 1:23 (69% relative time). Combined with triangle inequality, it enhances sphere tracing by 60% [S1].

### Adaptive Stepsize Rendering
A rendering method where the ray marching step size is provided per-pixel via an input tensor, allowing variable sampling density. The input `stepsize` tensor is of size $H \times W$ with positive values. The output tensor contains 8 channels: mask, normal (XYZ), depth, AO, and flow (XY). Unlike standard rendering, it accepts per-pixel step sizes, enabling adaptive sampling based on importance maps [S2, unofficial].

### Error Estimate
Calculates a relative error metric for adaptive stepping. The metric is computed as $r_v = |k / \max(|y|, \text{threshold})|$, and the final error is $\max(r_v.x, \max(r_v.y, r_v.z)) + \text{FLOAT\_MIN}$ [S3, unofficial].

### FBM Performance Optimization
Optimizations for Fractal Brownian Motion (FBM) used in procedural noise for volume rendering. Includes reducing octave count based on distance ($t$) using logarithmic scaling, multi-level LOD (e.g., 3, 9, 16 octaves), using texture sampling instead of hash functions, manually unrolling loops, and using directional derivatives instead of full gradients (1 sample vs 3). Early termination if accumulated amplitude exceeds 0.99 [S4].

## Background
Step size optimization evolved from the basic ray marching techniques used in early volume rendering and implicit surface rendering. Sphere tracing, introduced by [Kay & Kajiya, 1986], established the baseline of using the SDF value as the step size. Enhancements like the Triangle Inequality and Convexity were developed to reduce the computational cost of evaluating distances for complex scenes [S1]. In volume rendering, adaptive stepping emerged to handle the vast empty space in volumetric datasets, leveraging multi-resolution textures (mipmaps) and density thresholds to skip empty regions [S2, unofficial][S4]. Modern implementations often combine these geometric heuristics with temporal accumulation and GPU-specific optimizations like per-pixel tensor inputs [S2, unofficial].

## Sources

[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Learning adaptive sampling and reconstruction for Volume Visualization, https://github.com/shamanDevel/AdaptiveSampling, fetched 2026-10-27 (unofficial)
[S3] tink-expo/ntsl-graphics, https://github.com/tink-expo/ntsl-graphics, fetched 2026-10-26 (unofficial)
[S4] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
