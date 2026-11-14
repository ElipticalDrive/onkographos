# Path Tracing and Sampling


## Overview
Path Tracing and Sampling is a family of Monte Carlo algorithms that evaluate visibility and illumination at a point $\mathbf{p}$ by recursively tracing rays from the camera into the scene to simulate global illumination, including direct lighting, shadows, and indirect bounces [G]. It sets itself apart from local illumination methods by explicitly handling indirect light transport through recursive path generation, and from screen-space techniques by being view-independent and physically accurate [G].

## Shared principles

## Subtopics

### [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md)
A real-time technique for rendering participating media (fog, smoke, dust) by simulating light interaction with particles, assuming light scatters only once before reaching the camera. It solves the problem of realistic volumetric lighting and light shafts by integrating the volume rendering equation along camera rays, using shadow maps for occlusion and fixed-length stepping for accumulation. It differs from screen-space god ray techniques by handling multiple light sources, varying density, and physically correct behavior from any viewing angle, and differs from [Basic Volume Rendering and Ray Marching Loop with Absorption & Scattering](6821-basic-volume-rendering-and-ray-marching-loop-with-absorption.md) by explicitly detailing the shadow map sampling and throughput accumulation logic for heterogeneous media. Choose it when real-time performance is required for heterogeneous media with single-scattering approximations.

### [Sphere Light Sampling with Multiple Strategies](2207-sphere-light-sampling-with-multiple-strategies.md)
Calculates direct lighting from a spherical light source using Monte Carlo integration with various sampling strategies, including Importance Solid Angle, Importance Surface Area, Uniform Solid Angle, and others. It sets itself apart from siblings like [Dwivedi Sampling](5537-dwivedi-sampling.md) by explicitly handling the geometry of a spherical light source through multiple distinct probability density functions (PDFs) rather than a single fixed approach, allowing the renderer to choose the most efficient estimator for the light's position relative to the shading point. Choose it when rendering scenes with prominent spherical area lights and varying distances to the shading point.

### [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md)
A backward ray tracing algorithm that simulates light transport by recursively tracing rays from the camera, accumulating color contributions from surface interactions until a maximum bounce count is reached or a probabilistic termination condition is met. It is distinguished from its siblings by its explicit management of recursion depth combined with Russian Roulette termination based on the maximum component of the surface albedo, contrasting with methods that rely solely on fixed depth limits or discrete event sampling for sparse media. Choose it when strict control over path depth and unbiased termination via albedo-based survival probability is required.

### [Unidirectional Path Tracing](2558-unidirectional-path-tracing.md)
A Monte Carlo method that evaluates light transport by tracing rays from the camera into the scene, accumulating radiance through a sequence of interactions with surfaces and participating media. It solves the problem of computing global illumination by recursively sampling the next event in the light transport path, handling both surface scattering (BSDF) and volumetric scattering (medium sampling) within a single iterative loop. Unlike [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md), which may separate the path construction logic more rigidly, or [Path Tracing Extension with MIS](3291-path-tracing-extension-with-mis.md), which explicitly combines multiple sampling strategies, standard unidirectional path tracing typically relies on direct light sampling or next-event estimation for direct illumination and simple Russian Roulette for path termination. It differs from [Volumetric Path Tracing](1863-volumetric-path-tracing.md) in that the latter explicitly extends the path tracer to handle complex multiple scattering in participating media using delta tracking, whereas the standard method often treats volumes as secondary interactions or uses simpler approximations unless extended. Choose it for standard global illumination pipelines requiring a unified loop for surface and volume interactions.

### [Path Tracing Extension with MIS](3291-path-tracing-extension-with-mis.md)
This method extends a path tracing kernel by marching rays through sparse voxel data, shading intersections using albedo and geometric normals, and generating new ray directions via Multiple Importance Sampling (MIS) between sun disk and cosine-weighted hemisphere sampling. It differs from standard path tracing by explicitly handling sparse voxel traversal for volume shading and decoupling the accumulation buffer from the pixel grid in its wavefront variant. Choose it when rendering sparse voxel environments with strong directional sunlight where variance reduction via MIS is critical.

### [Path Tracing with Volume Interaction](3320-path-tracing-with-volume-interaction.md)
A path tracing algorithm that handles both surface and participating media intersections, using Russian Roulette for termination and power heuristic for light sampling. It evaluates multiple scattering in participating media, where light takes random paths through the medium until reaching a point, analogous to indirect illumination in surface rendering. It builds on [Path Tracing and Sampling](2526-path-tracing-and-sampling.md) by integrating volume intersection logic and isotropic scattering models directly into the path generation loop. Choose it when accurate multiple scattering in participating media is required alongside surface interactions.

### [Motion Blur](6441-motion-blur.md)
Motion blur simulates the integration of light over a finite shutter interval $[t_0, t_1]$, resulting in a blurred image when objects or the camera move during exposure. In path tracing, this is achieved by assigning a unique random time sample to each ray, naturally accumulating motion blur across samples. It differs from post-processing blur by preserving correct geometric occlusion and depth-of-field interactions at the sampled time instances. Choose it when physically accurate motion blur is needed without post-processing artifacts.

### [Differentiable Shadow Kernel Sampling via Normal Distribution](8431-differentiable-shadow-kernel-sampling-via-normal-distributio.md)
This method enables differentiable path tracing by importance sampling shadow offsets using a Normal distribution kernel in parameter space. It allows for the computation of gradients with respect to scene parameters by treating shadowing as a convolution with a smooth kernel, rather than a hard binary visibility test. Unlike standard binary shadow mapping or hard-edged shadow volumes, this approach provides continuous visibility derivatives, which is essential for optimization tasks such as material parameter recovery or geometry refinement. Choose it when gradient-based optimization of scene parameters or geometry is required.

### [IADDF Path Tracing](9294-iaddf-path-tracing.md)
IADDF Path Tracing is a Monte Carlo rendering method that solves the classical rendering equation using Intrinsic Appearance Differentiable Distance Fields (IADDF). It traces rays backwards from the eye to light sources, utilizing the DDF to find surface intersections and material/lighting fields to compute radiance. It is distinguished from standard path tracing by its reliance on differentiable implicit fields for geometry and appearance, allowing for continuous, analytic surface representation and shading without explicit mesh topology. Choose it when working with implicit geometry representations or differentiable appearance fields.

### Tree-based Edge Sampling for Secondary Visibility
- [S1] A method for sampling edges for secondary visibility (shadow, reflection, global illumination) where the viewpoint is a shading point, analogous to many-lights sampling.
- [S1] Build a tree data structure over the set of geometric edges.
- [S1] For each node in the tree, collect summary statistics of the edges (e.g., total emission power, spatial bounding box, orientation ranges) to determine node importance.
- [S1] Traverse the tree to select edges, rejecting nodes that contain only non-silhouette edges relative to the current shading point.

### Light Transport via Path Reversibility and Density Sampling
- [S2] A method to calculate shading values for points in a scene by computing the probability of light rays from a source reaching the point and the probability of camera rays intersecting that point, leveraging light path reversibility.
- [S2] Select np points from the scene with high density.
- [S2] For a point light source at xl and a point x, sample ns points x1...xns along the path between xl and x, where xns = x.
- [S2] Compute q_ns by applying the same sampling procedure from the camera origin to x as used for the light source path.
- [S2] Utilize nc viewpoints with the lowest rate of missing values to obtain the shading vector in specific order.
- [S2] Obtain ns sampling points by projecting pixels from these viewpoints along camera rays, determining projection depth using formula 8.
- [S2] Apply post-processing to address holes in the shading map caused by occlusions and residual geometry details.
- [S2] Brightness formula: $b = p_{ns} \cdot \frac{I}{\|x - x_l\|^2}$, where $b$ is brightness, $p_{ns}$ is the probability of ray propagation to $x_{ns}$, $I$ is light intensity, and $\|x - x_l\|$ is the distance.
- [S2] Shading formula: $s = q_{ns} \cdot b$, where $s$ is the shading value, $q_{ns}$ is the probability of a camera ray intersecting the point, and $b$ is the brightness calculated previously.

## Background
Path tracing was introduced by James Kajiya in 1986 as a generalization of the rendering equation to handle complex global illumination effects [G]. The technique relies on Monte Carlo integration to estimate the integral of light transport, using Russian Roulette for unbiased termination of infinite path sequences [G]. It has become a standard method for offline photorealistic rendering and is increasingly used in real-time graphics via techniques like [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md) and [Path Tracing Extension with MIS](3291-path-tracing-extension-with-mis.md) [G].

## Sources
[S1] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S2] NeRF Synthesis with Shading Guidance, https://link.springer.com/chapter/10.1007/978-981-99-9666-7_16, fetched 2026-10-26
