# Path Tracing and Sampling

Also called: Path Tracing, Monte Carlo Path Tracing, Russian Roulette Path Tracing

## Overview
Path Tracing and Sampling is a family of Monte Carlo algorithms that solve the rendering equation by recursively tracing rays from the camera into the scene to simulate global illumination, including direct lighting, shadows, and indirect bounces [S2, unofficial][S1]. It sets itself apart from local illumination methods (like [Lighting Models](2056-lighting-models.md) or [Shadow Ray Casting](2796-shadow-ray-casting.md)) by explicitly handling indirect light transport through recursive path generation, and from screen-space techniques by being view-independent and physically accurate [S1].

## Shared principles
All methods in this family share the core mechanism of evaluating visibility or illumination at a specific point $\mathbf{p}$ within or on the boundary of a volume, based on a light source direction $\mathbf{l}$ or environment [G].

1.  **Visibility/Transmittance Evaluation:** The fundamental operation is determining how much light reaches $\mathbf{p}$ from a source [G]. This involves integrating the extinction coefficient $\sigma_t(\mathbf{x})$ along a path from the source to $\mathbf{p}$ (or vice versa) [G]. The transmittance $T$ is calculated as:
    $$ T = \exp\left(-\int_{\mathbf{p}_{source}}^{\mathbf{p}} \sigma_t(\mathbf{x}) \, d\mathbf{x}\right) $$
    In discrete implementations, this integral is approximated by summing $\sigma_t \cdot \Delta s$ over samples along the ray [G].

2.  **Local Shading Models:** For surface-like appearance in volumes, normals are approximated from the volume gradient $\mathbf{g} = \nabla V$ [G]. Usually, a simple absorption and emission model is used because it is relatively cheap to evaluate and gives good enough results [S1].

3.  **Shadowing Approximation:** Shadows are computed by testing occlusion along the direction to the light source [G].

4.  **Ambient Occlusion:** Indirect illumination is approximated by integrating over the hemisphere or using simplified models (e.g., upward ray for sky light) [G]. The ambient contribution $L_{ambient}$ is often modulated by a shadowing factor based on local density:
    $$ L_{ambient} = L_{sky} \cdot \exp(-D_{up}) \cdot AO_{factor} $$
    where $D_{up}$ is the integrated density along an upward ray [G].

## Subtopics

### [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md)
Single Scattering Volume Rendering with Ray Marching is a real-time technique for rendering participating media (fog, smoke, dust) by simulating light interaction with particles, assuming light scatters only once before reaching the camera. It solves the problem of generating volumetric light-shafts and self-shadowing in real-time by approximating the radiative transport equation, ignoring multiple scattering to avoid the computational cost of distant fog blurring. Unlike screen-space god-ray techniques which are limited to single visible lights and suffer from edge artifacts, this method handles multiple light sources, colored fog, varying density, and physically correct behavior from any viewing angle by stepping along each camera ray and sampling shadow maps. It builds on [Path Tracing and Sampling](2526-path-tracing-and-sampling.md) by applying local illumination principles to volume segments.

### [Sphere Light Sampling with Multiple Strategies](2207-sphere-light-sampling-with-multiple-strategies.md)
Calculates direct lighting from a spherical light source using Monte Carlo integration with various sampling strategies, including Importance Solid Angle, Importance Surface Area, Uniform Solid Angle, and others. It sets itself apart from siblings like [Dwivedi Sampling](5537-dwivedi-sampling.md) by explicitly handling the geometry of a spherical light source through multiple distinct probability density functions (PDFs) rather than a single fixed approach, allowing the renderer to choose the most efficient estimator for the light's position relative to the shading point.

### [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md)
Explicit Path Tracing with Russian Roulette is a backward ray tracing algorithm that simulates light transport by recursively tracing rays from the camera, accumulating color contributions from surface interactions until a maximum bounce count is reached or a probabilistic termination condition is met. It is distinguished from its siblings by its explicit management of recursion depth combined with Russian Roulette termination based on the maximum component of the surface albedo, contrasting with methods that rely solely on fixed depth limits or discrete event sampling for sparse media.

### [Unidirectional Path Tracing](2558-unidirectional-path-tracing.md)
Unidirectional Path Tracing is a Monte Carlo method for evaluating visibility and illumination at points within or on the boundary of a volume, tracing rays from the camera into the scene. It handles surface interactions (BSDF sampling) and volumetric scattering within a single loop, terminated by Russian Roulette. Unlike explicit path tracing with Russian Roulette which separates the termination logic into a distinct post-loop phase, this method integrates survival probability checks directly into the main traversal loop to handle path termination dynamically.

### [Path Tracing Extension with MIS](3291-path-tracing-extension-with-mis.md)
This method extends a path tracing kernel by marching rays through sparse voxel data, shading intersections using albedo and geometric normals, and generating new ray directions via Multiple Importance Sampling (MIS) between sun disk and cosine-weighted hemisphere sampling. It differs from standard path tracing by decoupling the accumulation buffer from the pixel grid (via spatial hashing in its wavefront variant) and explicitly handling sparse voxel traversal for volume shading.

### [Path Tracing with Volume Interaction](3320-path-tracing-with-volume-interaction.md)
Path Tracing with Volume Interaction is a path tracing algorithm that handles both surface and participating media intersections, using Russian Roulette for termination and power heuristic for light sampling. It evaluates multiple scattering in participating media, where light takes random paths through the medium until reaching a point, analogous to indirect illumination in surface rendering. It builds on [Path Tracing and Sampling](2526-path-tracing-and-sampling.md) by integrating volume intersection logic and isotropic scattering models directly into the path generation loop.

### [Dwivedi Sampling](5537-dwivedi-sampling.md)
Dwivedi Sampling is a variance reduction technique for path tracing in participating media, specifically designed for high-scattering, isotropic environments. It addresses the problem of "lost paths" where random walks become trapped deep within dense volumes, failing to contribute to the final image. Unlike standard distance sampling which treats the medium as infinite or uniform, Dwivedi Sampling biases the random walk to exit the medium as quickly as possible by approximating the local geometry as a slab. This approach leverages the principle that paths near the boundary have higher contribution than those deep within the volume. It is distinct from [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md) by explicitly modeling the exit probability of the medium, and from [Sphere Light Sampling with Multiple Strategies](2207-sphere-light-sampling-with-multiple-strategies.md) by focusing on geometric escape rather than light source visibility.

### [Motion Blur](6441-motion-blur.md)
Motion blur simulates the integration of radiance over a finite shutter interval $[t_0, t_1]$ rather than at a single instant. In path tracing, this is achieved by assigning a unique random time sample to each ray, causing moving objects to appear blurred along their trajectories. It differs from post-processing blur by correctly handling occlusion and perspective changes during the exposure.

### [Differentiable Shadow Kernel Sampling via Normal Distribution](8431-differentiable-shadow-kernel-sampling-via-normal-distributio.md)
This method enables differentiable path tracing by importance sampling shadow offsets using a Normal distribution kernel in parameter space. It allows for the computation of gradients with respect to scene parameters by treating shadowing as a convolution with a smooth kernel, rather than a hard binary visibility test. Unlike standard binary shadow mapping or hard-edged shadow volumes, this approach provides continuous visibility derivatives, which is essential for optimization tasks such as material parameter recovery or geometry refinement.

### [IADDF Path Tracing](9294-iaddf-path-tracing.md)
A Monte Carlo rendering method solving the classical rendering equation using the IADDF. It traces rays backwards from the eye to light sources, using the DDF to find surface intersections and the material/lighting fields to compute radiance.

### Tree-based Edge Sampling for Secondary Visibility
- [S1] A method for sampling edges for secondary visibility (shadow, reflection, global illumination) where the viewpoint is a shading point, analogous to many-lights sampling.
- [S1] Build a tree data structure over the set of geometric edges.
- [S1] For each node in the tree, collect summary statistics of the edges (e.g., total emission power, spatial bounding box, orientation ranges) to determine node importance.
- [S1] Traverse the tree to select edges, rejecting nodes that contain only non-silhouette edges relative to the current shading point.

## Background
Path tracing was introduced by James Kajiya in 1986 as a generalization of the rendering equation to handle complex global illumination effects [G]. The technique relies on Monte Carlo integration to estimate the integral of light transport, using Russian Roulette for unbiased termination of infinite path sequences [S2, unofficial]. It has become a standard method for offline photorealistic rendering and is increasingly used in real-time graphics via techniques like [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md) and [Path Tracing Extension with MIS](3291-path-tracing-extension-with-mis.md).

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
