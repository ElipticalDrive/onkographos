# Step Size Optimization

Also called: Adaptive Ray Marching, Dynamic Step Sizing, Space Leaping

## Overview
Step Size Optimization determines the distance $\Delta s$ advanced along a ray $\mathbf{d}$ at each iteration of volume rendering or ray marching, replacing the naive constant step with a function $\Delta s = f(\mathbf{p}, \mathbf{d}, \text{context})$ [Parent]. It solves the trade-off between rendering accuracy (avoiding missed geometry or aliasing) and computational efficiency (reducing sample counts in empty or smooth regions) [Parent]. It sets itself apart from [Standard Constant Step Ray Marching](1080-standard-constant-step-ray-marching.md) by varying $\Delta s$ based on local data properties, and from [Sphere Tracing](1674-sphere-tracing.md) by not strictly relying on the Signed Distance Function (SDF) value as the sole upper bound for step size, though it may use SDFs as a heuristic [Parent].

## Shared principles
The core mechanism replaces the constant step $\Delta s = \text{const}$ with a function $\Delta s = f(\mathbf{p}, \mathbf{d}, \text{context})$, where $\mathbf{p}$ is the current ray position, $\mathbf{d}$ is the normalized ray direction, and context includes previous samples, camera state, or dataset metadata [Parent]. The general update rule for a ray origin $\mathbf{o}$ and direction $\mathbf{d}$ is:
$$ \mathbf{p}_{i+1} = \mathbf{p}_i + \Delta s_i \mathbf{d} $$
$$ t_{i+1} = t_i + \Delta s_i $$

Key shared constraints and goals include:
1.  **Safety/Correctness:** The step size must not cause the ray to "overshoot" or "penetrate" geometry or density features that should have been sampled. For SDFs, this often requires $\Delta s \le d(\mathbf{p})$ (Sphere Tracing) or a conservative fraction thereof. For volumes, it requires sufficient sampling density to satisfy the Nyquist criterion or to capture density gradients accurately [Parent].
2.  **Efficiency:** The step size should increase in empty or low-gradient regions to reduce sample count [Parent].
3.  **Termination:** Step size changes interact with termination rules (e.g., Early Ray Termination). A larger step size might skip the exact intersection point, requiring post-processing refinement (e.g., binary search) [Parent].

## Subtopics

### [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md)
Guarantees no overshoot by clamping SDF values and applying a conservative ratio, distinct from standard sphere tracing by explicitly handling distortion artifacts.

### [Adaptive Step Size Ray Marching](2312-adaptive-step-size-ray-marching.md)
Uses GPU-based temporal accumulation and noise buffers to drive continuous step size variation based on density and distance, distinct from safe methods by its focus on noise management and from gap-limited methods by continuous scaling.

### [Dynamic Sampling Step Adjustment](2442-dynamic-sampling-step-adjustment.md)
Accelerates rendering by doubling step size and coarsening mipmap levels in empty regions, distinct from safe methods by using density thresholds rather than SDFs, and from adaptive methods by using discrete doubling.

### [Compute Minimal Sample Distance Per Pixel](2684-compute-minimal-sample-distance-per-pixel.md)
Computes the minimal sample distance by projecting dataset cell axes onto the viewport, serving as a lower bound for step size, distinct by focusing on angular voxel spread rather than local gradients.

### [Error Estimate](2768-error-estimate.md)
Calculates relative error based on distance to an isovalue, distinct by introducing data-dependence on the isovalue parameter to reduce computation far from the surface.

### [Gap-Limited Ray Marching](4412-gap-limited-ray-marching.md)
Prevents overshoot on heightfields by capping step size at half the vertical clearance, distinct by relying on vertical gap bounds rather than derivatives or global SDFs.

### Triangle Inequality Enhancement for Sphere Tracing
- **Definition:** An optimization that eliminates unnecessary distance computations for collections of objects by using the triangle inequality to bound the distance to an object based on previous evaluations and ray traversal [S1]. It is particularly effective for large assortments of objects [S1].
- **Pitfall:** If all primitives (e.g., 14 Bezier curves) are nearly equidistant from the eye, the triangle inequality does not significantly reduce unnecessary distance evaluations until sphere tracing has traversed much of each ray [S1].
- **Performance:** For 9 spheres/plane, it reduced execution time from 2:53 to 1:19 (46% relative time) [S1]. For the zeno logo, it reduced time from 26:29 to 19:23 (73% relative time) [S1]. When combined with convexity, it enhances ordinary sphere tracing by 60% [S1].

### Convexity Enhancement
- **Definition:** Uses knowledge of convexity to increase step size along the ray. For a convex set $A$, the line segment from the current point $\mathbf{p}$ to the tangent plane intersection does not intersect the surface except possibly at the endpoint [S1]. The step is defined as $\text{lerp}(\mathbf{p}, \mathbf{p} + d(\mathbf{p}) \frac{\mathbf{n}(\mathbf{p})}{\|\mathbf{n}(\mathbf{p})\|}) = \mathbf{p} + d(\mathbf{p}) \mathbf{v}$ [S1]. If $d(\mathbf{p}) < 0$, the ray does not intersect the implicit surface [S1].
- **Pitfall:** Rays parallel to a horizon line (e.g., ground plane) would step at fixed intervals looking for an intersection that never happens. Corollary 4.1 avoids this by detecting if the ray has stepped beyond the object [S1].
- **Performance:** In the zeno logo test, it reduced execution time from 26:29 to 19:23 (73% relative time) [S1]. For a single sphere, it reduced time from 2:00 to 1:23 (69% relative time) [S1].
- **Comparison:** Convexity enhancement causes sphere tracing to converge quadratically, similar to Newton's method [S1]. It provides a faster order of convergence, confirmed by the slope on the left of the step-size histogram, whereas standard sphere tracing is linearly convergent [S1].

### Adaptive Stepsize Rendering
- **Definition:** A rendering method where the ray marching step size is provided per-pixel via an input tensor, allowing for variable sampling density across the image [S2, unofficial].
- **Step:** The PyTorch wrapper `adaptiveStepsizeRendering` asserts that the input `stepsizes` tensor has shape B*1*H*W and C=1 [S2, unofficial]. The stepsizes tensor is moved to the CUDA device [S2, unofficial]. For each batch item, the renderer settings are sent to update the renderer state [S2, unofficial]. The viewport dimensions are extracted from settings and asserted to match the H and W of the stepsizes tensor [S2, unofficial]. The stepsizes tensor is padded to match the viewport requirements [S2, unofficial].
- **Parameter:** `fd_epsilon`: A float parameter with default value $10^{-5}$, used for gradient propagation support [S2, unofficial].
- **Layout:** Input tensor `stepsize` is a float tensor on the GPU of size HxW with only positive values [S2, unofficial]. Output tensor `output` is a float tensor on the GPU of size BxHxW with B=8 channels: 0=mask, 1=normalX, 2=normalY, 3=normalZ, 4=depth, 5=ao, 6=flowX, 7=flowY [S2, unofficial].
- **Comparison:** Unlike standard rendering which uses a uniform or pre-computed step size, this method accepts a per-pixel tensor of step sizes, enabling adaptive sampling based on importance maps [S2, unofficial].

### FBM Performance Optimization
- **Step:** Reduce octave count based on distance or time parameter $t$ using logarithmic scaling [S3]. Use multi-level LOD with different octave counts for different resolutions (e.g., 3, 9, 16 octaves) [S3]. Use texture sampling instead of mathematical hash functions for noise generation [S3]. Manually unroll small loops and vary lacunarity slightly [S3]. Use adaptive step size based on distance or time [S3]. Use directional derivative instead of full gradient to reduce sample count from 3 to 1 [S3]. Terminate loop early if accumulated amplitude exceeds 0.99 [S3].

### Adaptive Step Size for Interaction
- **Definition:** Increasing the step size in ray casting or reducing the number of slices reduces computational effort but can introduce artifacts [S4, unofficial].
- **Step:** During interaction (e.g., changing viewpoint), increase the step size for fluent interaction. After interaction, produce a high-quality still image with standard parameters [S4, unofficial].
- **Usage:** Described in Hadwiger and Rezk Salama, 2004 [S4, unofficial].

## Background
The concept of varying step size in ray marching originates from the need to optimize the performance of implicit surface rendering and volume visualization. Sphere tracing, introduced by Kay and Kajiya in 1986, established the baseline of using the SDF value as the step size [S1]. Subsequent work, such as the enhancements by Hoffmann et al. (1990) on convexity and triangle inequality, demonstrated that geometric properties could be leveraged to increase step sizes safely [S1]. In volume rendering, adaptive stepping became crucial for handling sparse data and temporal coherence, with methods like those described by Hadwiger and Rezk Salama (2004) advocating for dynamic step adjustment during interaction [S4, unofficial]. Modern implementations, such as those using PyTorch for adaptive reconstruction, further generalize this by allowing per-pixel step size tensors [S2, unofficial].

## Sources
[S1] Sphere Tracing: A Geometric Method for the Antialiased Ray ..., https://www.researchgate.net/publication/2792108_Sphere_Tracing_A_Geometric_Method_for_the_Antialiased_Ray_Tracing_of_Implicit_Surfaces, fetched 2026-10-27
[S2] Learning adaptive sampling and reconstruction for Volume Visualization, https://github.com/shamanDevel/AdaptiveSampling, fetched 2026-10-27 (unofficial)
[S3] skills/skills/shader-dev/techniques/procedural-noise.md at main ..., https://github.com/MiniMax-AI/skills/blob/main/skills/shader-dev/techniques/procedural-noise.md, fetched 2026-10-27
[S4] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
