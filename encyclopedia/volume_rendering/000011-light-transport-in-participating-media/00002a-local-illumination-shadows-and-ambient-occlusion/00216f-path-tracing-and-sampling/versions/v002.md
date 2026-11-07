# Path Tracing and Sampling
## Overview
Path tracing recursively traces rays from the camera into the scene to simulate light transport, accumulating radiance contributions from surface reflections and volumetric scattering [G]. It solves the global illumination problem by stochastically evaluating the rendering equation without relying on precomputed approximations [G]. What sets it apart from its siblings is its unified recursive path construction: unlike [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md) which uses fixed-step integration and ignores multiple scattering, or [Sphere Light Sampling with Multiple Strategies](2207-sphere-light-sampling-with-multiple-strategies.md) which focuses on direct illumination from specific light geometries, path tracing dynamically adapts path length and sampling strategies based on local material properties and scene geometry.
## Shared principles
All methods in this family share the core mechanism of evaluating visibility or illumination at a specific point $\mathbf{p}$ within or on the boundary of a volume, based on a light source direction $\mathbf{l}$ or environment [G].

1.  **Visibility/Transmittance Evaluation:** The fundamental operation is determining how much light reaches $\mathbf{p}$ from a source [G]. This involves integrating the extinction coefficient $\sigma_t(\mathbf{x})$ along a path from the source to $\mathbf{p}$ (or vice versa) [G]. The transmittance $T$ is calculated as:
    $$ T = \exp\left(-\int_{\mathbf{p}_{source}}^{\mathbf{p}} \sigma_t(\mathbf{x}) \, d\mathbf{x}\right) $$
    In discrete implementations, this integral is approximated by summing $\sigma_t \cdot \Delta s$ over samples along the ray [G].

2.  **Local Shading Models:** For surface-like appearance in volumes, normals are approximated from the volume gradient $\mathbf{g} = \nabla V$ [G]. Usually, a simple absorption and emission model is used because it is relatively cheap to evaluate and gives good enough results [G].

3.  **Shadowing Approximation:** Shadows are computed by testing occlusion along the direction to the light source [G].

4.  **Ambient Occlusion:** Indirect illumination is approximated by integrating over the hemisphere or using simplified models (e.g., upward ray for sky light) [G]. The ambient contribution $L_{ambient}$ is often modulated by a shadowing factor based on local density:
    $$ L_{ambient} = L_{sky} \cdot \exp(-D_{up}) \cdot AO_{factor} $$
    where $D_{up}$ is the integrated density along an upward ray [G].
## Subtopics

### [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md)
Single Scattering Volume Rendering with Ray Marching is a GPU-based algorithm that computes volumetric light-shafts in participating media by assuming light scatters only once before reaching the camera. It solves the radiative transport equation by ignoring multiple scattering, which allows the in-scattering term to be computed independently of unknown radiance. Unlike screen-space radial blur methods, this technique handles multiple light sources, colored fog, varying density, and physically correct behavior from any viewing angle, though it is more computationally expensive. Choose it when single-scattering approximation is sufficient and GPU performance is prioritized.

### [Sphere Light Sampling with Multiple Strategies](2207-sphere-light-sampling-with-multiple-strategies.md)
This method estimates direct illumination from a spherical light source using Monte Carlo integration, supporting multiple sampling strategies including Importance Solid Angle, Importance Surface Area, Uniform Solid Angle, Uniform Hemisphere Solid Angle, Importance Phong, and Cosine Weighted sampling. It differs from its siblings by providing a unified framework for shading against a spherical light, allowing the renderer to switch between strategies based on the geometric relationship between the shading point and the light, or to balance variance and bias. Choose it when rendering scenes with prominent spherical lights and needing flexible sampling trade-offs.

### [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md)
Explicit Path Tracing is a rendering algorithm that simulates light transport by recursively tracing rays from the camera into the scene, accumulating color contributions from surface interactions until a maximum bounce count is reached or a termination condition is met. It explicitly manages recursion depth and uses Russian Roulette to terminate paths probabilistically based on surface albedo. Unlike its sibling [Path Tracing with Discrete Event Sampling](5168-path-tracing-with-discrete-event-sampling.md), which replaces continuous termination with discrete probabilities for sparse media, this method relies on fixed depth limits and probabilistic survival based on material reflectance. Choose it for standard surface-based global illumination where explicit bounce control is preferred.

### [Unidirectional Path Tracing](2558-unidirectional-path-tracing.md)
Unidirectional Path Tracing is a Monte Carlo method for evaluating visibility and illumination at points within or on the boundary of a volume, tracing rays from the camera into the scene. It handles surface interactions (BSDF sampling) and volumetric scattering within a single loop, terminated by Russian Roulette. Unlike explicit path tracing with Russian Roulette which separates the termination logic into a distinct post-loop phase, this method integrates survival probability checks directly into the main traversal loop to handle path termination dynamically. Choose it for real-time or memory-constrained applications where a single path tree is sufficient.

### [Path Tracing Extension with MIS](3291-path-tracing-extension-with-mis.md)
This method extends a path tracing kernel by marching rays through sparse voxel data, shading intersections using albedo and geometric normals, and generating new ray directions via Multiple Importance Sampling (MIS) between sun disk and cosine-weighted hemisphere sampling. It differs from standard path tracing by decoupling the accumulation buffer from the pixel grid (via spatial hashing in its wavefront variant) and explicitly handling sparse voxel traversal for volume shading. Choose it when rendering large-scale outdoor scenes with sparse voxel representations and requiring efficient MIS between direct and indirect lighting.

### [Path Tracing with Volume Interaction](3320-path-tracing-with-volume-interaction.md)
Path Tracing with Volume Interaction is a global illumination algorithm that handles both surface and participating media intersections within a single unified framework, using Russian Roulette for path termination and power heuristic for light sampling. It solves the problem of simulating multiple scattering in participating media, where light takes random paths through the medium until reaching a point, analogous to indirect illumination in surface rendering. It sets itself apart from [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md) by supporting multiple scattering events rather than limiting interactions to the first scattering event, and from [Explicit Path Tracing with Russian Roulette](2209-explicit-path-tracing-with-russian-roulette.md) by explicitly integrating volume intersection logic and isotropic scattering models into the path generation loop. Choose it for scenes requiring accurate multiple scattering in participating media.

### [Dwivedi Sampling](5537-dwivedi-sampling.md)
#### definition
A technique to improve convergence by biasing the random walk to increase the chance of returning to the entry side or exiting the other side, reducing lost paths [G].
#### step
Shoot a probe ray to estimate the thickness of the object, crudely approximating the object as a slab [G]. Mix classical distance/direction sampling with distance and direction sampling methods [G]. Apply MIS between multiple methods using modified heuristic weights for distance and direction sampling [G].
#### parameter
The MIS weighting between classical and Dwivedi sampling is based on heuristic weights modified for distance and direction sampling [G].

### Motion Blur
#### definition
Simulates motion blur by opening the shutter for a time interval $[t_0, t_1]$ [S2]. In path tracing, each ray sample uses a different time, resulting in natural motion blur [S2].
#### step
Sample a random time within the shutter interval: $t = \text{lerp}(t_0, t_1, \text{random}())$ [S2]. Evaluate object transforms at the sampled time $t$ [S2].
#### parameter
Requires per-object velocity data or interpolated transforms to evaluate positions at time $t$ [S2].

### Tree-based Edge Sampling for Secondary Visibility
#### definition
A method for sampling edges for secondary visibility (shadow, reflection, global illumination) where the viewpoint is a shading point, analogous to many-lights sampling [S2].
#### step
Build a tree data structure over the set of geometric edges [S2]. For each node in the tree, collect summary statistics of the edges (e.g., total emission power, spatial bounding box, orientation ranges) to determine node importance [S2]. Traverse the tree to select edges, rejecting nodes that contain only non-silhouette edges relative to the current shading point [S2].

### IADDF Path Tracing
#### definition
A Monte Carlo rendering method solving the classical rendering equation using the IADDF. It traces rays backwards from the eye to light sources, using the DDF to find surface intersections and the material/lighting fields to compute radiance [S3].
#### step
For a given pixel and camera, cast a ray into the scene to find the scene point $q$ and normal $n$ via the DDF [S3]. Based on the importance sampler $\Psi$, sample new outgoing directions from the hemisphere about $q$ [S3]. Use the DDF to find the next surface in the sampled direction to continue the light path (bounce) [S3]. Continue bouncing until the ray misses any local geometry ($\xi = 0$), at which point return the value from the environment light $EL$ in that direction [S3]. Compute the final sampled pixel irradiance as the environment value attenuated by the material BRDF $f_B$ through the bounces, plus any emissions from $ML$ encountered along the way [S3].
#### code
Algorithm 1: PathTracePixelDDF(u, Π) Input: Pixel u and Camera Π Models: DDF (d, ξ), Appearance (fB, Ψ), and Illumination (ML, EL) Hyper-Parameters: Maximum bounces n, Number of MC Samples m Output: The pixel colour c at u [S3].

### [Differentiable Shadow Kernel Sampling via Normal Distribution](8431-differentiable-shadow-kernel-sampling-via-normal-distributio.md)
#### definition
A method for importance sampling shadow offsets using a Normal distribution kernel in parameter space to enable differentiable path tracing [G].

## Background
Not given in the sources. The method is widely used in both offline and real-time rendering pipelines for global illumination [G].
## Sources
[S1] Camera Model in Path Tracing - Pinhole, Lens, and DOF Guide | Dev ..., https://code-note-vr.vercel.app/pathtracer-learning-camera-model, fetched 2026-10-29 (unofficial)
[S2] Physics-based differentiable rendering, https://doi.org/10.1145/3388769.3407454, fetched 2026-10-29
[S3] Probabilistic Directed Distance Fields for Ray-Based Shape Representations, https://doi.org/10.1109/tpami.2025.3594225, fetched 2026-10-29
