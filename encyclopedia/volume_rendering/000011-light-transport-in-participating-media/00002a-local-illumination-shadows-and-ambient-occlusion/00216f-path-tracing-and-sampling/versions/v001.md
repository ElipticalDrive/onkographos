# Path Tracing and Sampling
## Overview
Path tracing is a Monte Carlo method for solving the rendering equation by recursively tracing rays from the camera into the scene, accumulating radiance through surface reflections and volumetric scattering [S1, S2, unofficial]. It solves the problem of global illumination by simulating the random walk of light, handling complex interactions like caustics, soft shadows, and subsurface scattering without precomputed approximations [S2, unofficial]. What sets it apart from its siblings is its unified recursive structure: unlike [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md) which approximates light transport via fixed-step integration, or [Light Sampling for NEE](2616-light-sampling-for-nee.md) which focuses on specific direct light contributions, path tracing constructs a continuous path tree (or single path in unidirectional variants) that dynamically adapts to scene geometry and material properties [S2, unofficial].
## Shared principles
All methods in this family share the core mechanism of evaluating visibility or illumination at a specific point $\mathbf{p}$ within or on the boundary of a volume, based on a light source direction $\mathbf{l}$ or environment.

1.  **Visibility/Transmittance Evaluation:** The fundamental operation is determining how much light reaches $\mathbf{p}$ from a source. This involves integrating the extinction coefficient $\sigma_t(\mathbf{x})$ along a path from the source to $\mathbf{p}$ (or vice versa). The transmittance $T$ is calculated as:
    $$ T = \exp\left(-\int_{\mathbf{p}_{source}}^{\mathbf{p}} \sigma_t(\mathbf{x}) \, d\mathbf{x}\right) $$
    In discrete implementations, this integral is approximated by summing $\sigma_t \cdot \Delta s$ over samples along the ray [S1, unofficial][S6, unofficial].

2.  **Local Shading Models:** For surface-like appearance in volumes, normals are approximated from the volume gradient $\mathbf{g} = \nabla V$. The shading contribution $C_{shade}$ is computed using standard local illumination models (e.g., Lambertian, Phong) [S1, unofficial]:
    $$ C_{shade} = k_d (\mathbf{n} \cdot \mathbf{l}) C_{diffuse} + k_s (\mathbf{n} \cdot \mathbf{h})^\alpha C_{specular} $$
    where $\mathbf{n} = \mathbf{g}/\|\mathbf{g}\|$ [S1, unofficial].

3.  **Shadowing Approximation:** Shadows are computed by testing occlusion along the direction to the light source. This can be done via ray casting (checking for density/geometry along the path) or via precomputed buffers (shadow maps, illumination buffers) [S1, unofficial][S2, unofficial].

4.  **Ambient Occlusion:** Indirect illumination is approximated by integrating over the hemisphere or using simplified models (e.g., upward ray for sky light). The ambient contribution $L_{ambient}$ is often modulated by a shadowing factor based on local density:
    $$ L_{ambient} = L_{sky} \cdot \exp(-D_{up}) \cdot AO_{factor} $$
    where $D_{up}$ is the integrated density along an upward ray [S1, unofficial][S2, unofficial].
## Subtopics

### [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md)
Single Scattering Volume Rendering with Ray Marching is a GPU-based algorithm that computes volumetric light-shafts (god rays) in participating media by assuming light scatters only once before reaching the camera. It solves the radiative transport equation by ignoring multiple scattering, allowing the in-scattering term to be computed independently of unknown radiance. Unlike screen-space radial blur methods, this technique handles multiple light sources, colored fog, varying density, and physically correct behavior from any viewing angle, though it is more computationally expensive.

### [Sphere Light Sampling with Multiple Strategies](2207-sphere-light-sampling-with-multiple-strategies.md)
This method estimates direct illumination from a spherical light source using Monte Carlo integration, supporting multiple sampling strategies including Importance Solid Angle, Importance Surface Area, Uniform Solid Angle, and others. It differs from [Importance Solid Angle Sampling](2236-sphere-light-shading-with-importance-solid-angle-sampling.md) by providing a unified framework that switches between geometric area sampling and solid angle sampling, and from [Importance Sampling for Sphere Surface Area](2223-importance-sampling-for-sphere-surface-area.md) by explicitly handling the probability density function (PDF) normalization for each strategy within a single estimator loop.

### [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md)
Explicit Path Tracing is a rendering algorithm that simulates light transport by recursively tracing rays from the camera into the scene, accumulating color contributions from surface interactions until a maximum bounce count is reached or a termination condition is met. It explicitly manages recursion depth and uses Russian Roulette to terminate paths probabilistically based on surface albedo. Unlike its sibling [Path Tracing with Discrete Event Sampling](5168-path-tracing-with-discrete-event-sampling.md), which replaces continuous termination with discrete probabilities for sparse media, this method relies on fixed depth limits and probabilistic survival based on material reflectance.

### [Unidirectional Path Tracing](2558-unidirectional-path-tracing.md)
Unidirectional Path Tracing is a Monte Carlo method for solving the rendering equation by tracing paths from the camera into the scene, accumulating radiance through surface reflections and volumetric scattering. It differs from bidirectional methods by constructing a single path tree rooted at the sensor, making it suitable for real-time or interactive applications where memory bandwidth for multiple path trees is prohibitive. Unlike explicit path tracing with Russian Roulette which separates the termination logic into a distinct post-loop phase, this method integrates survival probability checks directly into the main traversal loop to handle path termination dynamically.

### [Path Tracing Extension with MIS](3291-path-tracing-extension-with-mis.md)
A path tracing kernel that extends rays by marching them through sparse voxels, shading intersections using albedo and normal, and generating new ray directions using Multiple Importance Sampling (MIS) between sun disk and cosine-weighted hemisphere sampling. It differs from standard path tracing by decoupling the accumulation buffer from the pixel grid, enabling spatial averaging of radiance estimates before final display.

### [Path Tracing with Volume Interaction](3320-path-tracing-with-volume-interaction.md)
Path Tracing with Volume Interaction is a unified path tracing algorithm that handles both surface and participating media intersections within a single loop, using Russian Roulette for path termination and power heuristic for light sampling. It distinguishes itself from [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md) by supporting multiple bounces and indirect illumination through recursive path extension, while avoiding the complexity of [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md) by integrating volume scattering directly into the main traversal logic rather than treating it as a separate pass.

### Dwivedi Sampling
#### definition
- [S1, unofficial] A technique to improve convergence by biasing the random walk to increase the chance of returning to the entry side or exiting the other side, reducing lost paths.
#### step
- [S1, unofficial] Shoot a probe ray to estimate the thickness of the object, crudely approximating the object as a slab.
- [S1, unofficial] Mix classical distance/direction sampling with [Kd14] distance and direction sampling methods.
- [S1, unofficial] Apply MIS between 6 methods (for RGB renders) using modified heuristic weights from [Kd14] for distance and direction sampling.
#### parameter
- [S1, unofficial] The MIS weighting between classical and dwivedi is based on the heuristic from [Kd14], but modified to be slightly different for distance and direction sampling.

### Motion Blur
#### definition
- [S2, unofficial] Simulates motion blur by opening the shutter for a time interval $[t_0, t_1]$.
- [S2, unofficial] In path tracing, each ray sample uses a different time, resulting in natural motion blur.
#### step
- [S2, unofficial] Sample a random time within the shutter interval: $t = \text{lerp}(t_0, t_1, \text{random}())$.
- [S2, unofficial] Evaluate object transforms at the sampled time $t$.
#### parameter
- [S2, unofficial] Requires per-object velocity data or interpolated transforms to evaluate positions at time $t$.

## Background
Path tracing was introduced by James Kajiya in 1986 in his paper "The Rendering Equation" [S2, unofficial]. It provides a general framework for rendering by interpreting the rendering equation as a particle transport problem. The method has become the gold standard for offline rendering due to its ability to handle complex global illumination effects physically accurately. Variants like unidirectional path tracing are widely used in real-time graphics for their memory efficiency compared to bidirectional methods [S2, unofficial]. The integration of volume interaction allows the method to handle participating media, extending its applicability to fog, smoke, and subsurface scattering [S1, unofficial].
## Sources
[S1] Path Traced Subsurface Scattering Model, https://www.scribd.com/document/965284472/Pixar-Path-Traced-SSS-Paper, fetched 2026-10-29 (unofficial)
[S2] Camera Model in Path Tracing - Pinhole, Lens, and DOF Guide | Dev ..., https://code-note-vr.vercel.app/pathtracer-learning-camera-model, fetched 2026-10-29 (unofficial)
