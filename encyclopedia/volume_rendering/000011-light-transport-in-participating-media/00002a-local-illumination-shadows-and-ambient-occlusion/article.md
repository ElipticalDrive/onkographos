# Local illumination, shadows and ambient occlusion

Also called: Local shading, direct illumination in volumes, volumetric shadowing

## Overview
This family covers techniques for computing direct visibility and local shading in volume rendering, addressing the problem of how light from sources interacts with participating media and opaque geometry to produce shadows, self-shadowing, and surface-like orientation cues [G]. It distinguishes itself from siblings like [Optical models](33-optical-models.md) by focusing on the geometric and visibility aspects of light transport (shadows, occlusion) rather than the physical scattering phase functions, and from [Single and multiple scattering](35-single-and-multiple-scattering.md) by typically ignoring or approximating indirect bounces [G]. It differs from [Real-time approximations](36-real-time-approximations.md) in that it includes both offline precomputation methods (like Light Propagation Volumes) and real-time ray-marching techniques [G].

## Shared principles
All methods in this family share the core mechanism of evaluating visibility or illumination at a specific point $\mathbf{p}$ within or on the boundary of a volume, based on a light source direction $\mathbf{l}$ or environment [G].

1.  **Visibility/Transmittance Evaluation:** The fundamental operation is determining how much light reaches $\mathbf{p}$ from a source [G]. This involves integrating the extinction coefficient $\sigma_t(\mathbf{x})$ along a path from the source to $\mathbf{p}$ (or vice versa) [G]. The transmittance $T$ is calculated as:
    $$ T = \exp\left(-\int_{\mathbf{p}_{source}}^{\mathbf{p}} \sigma_t(\mathbf{x}) \, d\mathbf{x}\right) $$
    In discrete implementations, this integral is approximated by summing $\sigma_t \cdot \Delta s$ over samples along the ray [G].

2.  **Local Shading Models:** For surface-like appearance in volumes, normals are approximated from the volume gradient $\mathbf{g} = \nabla V$ [G]. Usually, a simple absorption and emission model is used because it is relatively cheap to evaluate and gives good enough results [S1, unofficial].

3.  **Shadowing Approximation:** Shadows are computed by testing occlusion along the direction to the light source [G].

4.  **Ambient Occlusion:** Indirect illumination is approximated by integrating over the hemisphere or using simplified models (e.g., upward ray for sky light) [G]. The ambient contribution $L_{ambient}$ is often modulated by a shadowing factor based on local density:
    $$ L_{ambient} = L_{sky} \cdot \exp(-D_{up}) \cdot AO_{factor} $$
    where $D_{up}$ is the integrated density along an upward ray [G].

## Subtopics

### [Lighting Models](2056-lighting-models.md)
Lighting Models apply local illumination heuristics to scalar field data to enhance the perception of 3D structure and surface orientation. Unlike physically based light transport methods that solve the radiative transfer equation, these models modulate existing volume samples or pixel colors using diffuse and specular terms derived from the data gradient. This family distinguishes itself from siblings like [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by focusing on local shading rather than global shadow computation, and from [Path Tracing and Sampling](2526-path-tracing-and-sampling.md) by using deterministic or semi-analytical approximations instead of stochastic integration.

### [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md)
Two-Pass Shadow Casting is a numerical algorithm for computing shadows in volume rendering by explicitly accounting for the transparency of participating media between a light source and a surface point, and between that point and the viewpoint. It solves the problem of realistic shadowing in scenes containing smoke, fog, clouds, or turbid water, where standard opaque shadow techniques fail to capture the attenuation of light through the volume. Unlike recursive ray tracing, which sends discrete "shadow feelers" to the light source at each point along the primary ray, this method integrates the shadowing effect continuously along the ray. It is distinguished from its sibling [Shadow Ray Casting](2796-shadow-ray-casting.md) by its volumetric integration approach and from [Shadow Mapping](2934-shadow-mapping.md) by its direct handling of participating media density rather than depth buffers.

### [Ambient Light Approximation](2244-ambient-light-approximation.md)
Ambient Light Approximation simulates indirect sky illumination in participating media by integrating the entire sky hemisphere into a single upward-pointing light value, weighted by local cloud density to simulate self-shadowing. Unlike full multiple scattering or complex irradiance tables, it uses a single precalculated or sampled upward ray to approximate the diffuse fill light, preventing the underside of clouds from appearing unnaturally dark. It differs from [Ambient Occlusion via Geometry/Neighbors](5936-ambient-occlusion-via-geometry-neighbors.md) by relying on volumetric density rather than surface geometry, and from [Bruneton-style Sky Irradiance](#bruneton-style-sky-irradiance) by using a simplified single-ray model instead of a multi-dimensional lookup table.

### [Ray-marched Volumetric Lighting with Density Thresholding](2456-ray-marched-volumetric-lighting-with-density-thresholding.md)
This method precomputes light intensities and shadows within a 3D volume based on a light source direction, color, decay, and a density mask, storing the result in a 3D texture for use during rendering. It differs from real-time ray marching by performing the light transport calculation offline (or in a dedicated compute pass) to decouple lighting cost from scene complexity, using density thresholding to filter noise and simulate god rays.

### [Path Tracing and Sampling](2526-path-tracing-and-sampling.md)
Path Tracing and Sampling is a stochastic method for solving the rendering equation in participating media by simulating the random walk of light paths. It handles both direct and indirect illumination by recursively sampling scattering events and light sources. It differs from local illumination methods by accounting for multiple bounces and complex visibility, but at a significantly higher computational cost.

### [Light Sampling for NEE](2616-light-sampling-for-nee.md)
Light Sampling for NEE is a technique for estimating direct illumination in participating media by explicitly sampling light sources and their surfaces to generate connection points for path tracing. It solves the problem of noisy direct lighting estimates by concentrating samples on bright emitters rather than relying solely on unidirectional shadow rays. It differs from [Shadow Ray Casting](2796-shadow-ray-casting.md) by using probabilistic sampling with importance weighting and from [Direct Light Sampling](3337-direct-light-sampling.md) by focusing specifically on the geometry of light emission and the associated probability density functions (PDFs) required for unbiased Nearest Energy Estimation (NEE) in media.

### [Shadow Ray Casting](2796-shadow-ray-casting.md)
Shadow Ray Casting is a family of techniques that determine local visibility by casting rays from a point of interest toward light sources or into the environment to test for occlusion by geometry or participating media. It solves the problem of computing accurate shadows and ambient occlusion in volume rendering and surface shading, distinguishing itself from global illumination methods by evaluating direct visibility explicitly rather than through stochastic path exploration or precomputed light maps.

### [Shadow Mapping](2934-shadow-mapping.md)
Shadow Mapping is a technique to efficiently render shadows by rendering the scene from the light's point of view to generate a depth map (shadow map). It determines if a point is occluded by another object from the light's perspective. It differs from [Shadow Ray Casting](2796-shadow-ray-casting.md) by using pre-rendered depth buffers instead of per-pixel ray traversal, and from [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by relying on a single depth buffer comparison rather than geometric silhouette processing.

### [Direct Light Sampling](3337-direct-light-sampling.md)
Sampling a single light source to compute direct illumination, handling different light types and ensuring the path is not blocked by other geometry.

### [Terrain Shading with Direct and Ambient Light](4424-terrain-shading-with-direct-and-ambient-light.md)
Computes the final color of a terrain surface point by combining albedo, direct sunlight (modulated by shadow and cloud shadow), and hemispheric ambient light (sky and ground bounce modulated by AO).

### [Ambient Occlusion via Geometry/Neighbors](5936-ambient-occlusion-via-geometry-neighbors.md)
Computes ambient occlusion based on the geometric proximity of surrounding surfaces or neighbors, rather than volumetric density.

### Shading
- **Definition:** Shading aims to enhance appearance by modeling effects like shadows, light scattering, and absorption. It depends on the chosen optical model [S1, unofficial].
- **Parameter:** A simple absorption and emission model is usually used because it is relatively cheap to evaluate and gives good enough results [S1, unofficial].
- **Comparison:** Local illumination shades each sample independently, ignoring the rest of the scene, so it cannot produce shadows or scattering but is cheap to compute. Global lighting simulates any effect for better results but is expensive and seldom used in real time [S1, unofficial].

### Powder Sugar Effect
- **Definition:** Approximating less scattered light at cloud fringes by reducing scattered light in low density areas using a powder function [S2, unofficial].
- **Step:** Multiply the increment to scattered light by `powder(density * stepsize)` [S2, unofficial].
- **Code:**
  ```glsl
  uniform float powder_decay;

  float powder(float d)
  {
    return 1.0 - exp(-powder_decay * d);
  }
  ```
  [S2, unofficial]

### Light Propagation Volumes
- **Definition:** Simulates first bounce indirect illumination by discretizing the scene into 3D cells in a continuous grid, where each cell holds a directional distribution of radiance [S3, unofficial].
- **Step:** Inject lighting into cells containing directly lit surfaces, using reflective shadow maps (or other methods) to find these cells. Form a radiance distribution around the normal, facing away from the surface, with color based on surface material. Propagate light from cell to cell: each cell analyzes the radiance field of its neighbors and modifies its own based radiance coming from all directions [S3, unofficial].
- **Parameter:** Encoding: 2nd order spherical harmonics are used to encode the directional distribution of radiance in each cell. Propagation steps: The number of steps allowed controls the light propagation [S3, unofficial].

### Ambient Aperture Lighting
- **Definition:** A technique that approximates the shape of unoccluded regions above the shading point via a cone to estimate occlusion of an area-light [S3, unofficial].
- **Step:** The set of 3D unoccluded directions is modeled as a circular aperture [S3, unofficial].
- **Pitfall:** If the set of unoccluded directions does not resemble an ellipse or a circle, it can produce incorrect shadowing results [S3, unofficial].
- **Comparison:** This method presents a lower storage requirement than horizon maps [S3, unofficial].

### Lighting
- **Definition:** Implements day/night sun cycle with scattering, soft shadows, and ambient occlusion [S5, unofficial].
- **Step:** Implements day/night sun cycle with sunset scattering, sun disc, halo and stars. Generates single-sample golden-angle PCF soft shadows jittered with interleaved gradient noise (TAA accumulates the penumbra). Computes bit-test ambient occlusion bilinearly interpolated across the hit face [S5, unofficial].

### Shadow Ray Marching
- **Definition:** Calculates sunlight contribution by casting a shadow ray from the surface towards the sun. If the shadow ray hits an object, the point is in shadow [S6].
- **Step:** If SUN_ENABLED, calculate sunDir from sceneData. Calculate nDotL = max(dot(normal, sunDir), 0.0). If nDotL > 0, create shadowOrigin = position + normal * 0.001. March shadow ray using March(shadowOrigin, sunDir, 64). If shadowRay.status != 2 (no hit), sunContribution = SUN_COLOR * nDotL * sceneData.sunIntensity [S6].

### Cloud Shadow Rendering
- **Definition:** A pass that renders cloud shadows onto terrain and objects by rendering a sky plane from the light source's perspective [S7, unofficial].
- **Step:** Shadow intensity is determined by the coverage value in the Weather map [S7, unofficial].

### Uniform Update for Lighting and Camera
- **Definition:** Computes and sets shader uniforms for up to 4 lights, eye position, and eye look direction in object space [S8, unofficial].
- **Step:** Compute the inverse of the actor's world matrix. Convert light position to object space. Set uniforms `light_position` (array of 4 `fvec3`) and `light_enable` (array of 4 `int`). Compute eye look direction in object space [S8, unofficial].

## Background
Local illumination and shadowing in volume rendering originated with the need to add realism to direct volume rendering (DVR) techniques, which initially relied on simple transfer functions without shading [S1, unofficial]. Early work by Max [Max 1995] established the distinction between local and global illumination in the context of volume rendering, noting that local methods are cheap but lack shadows, while global methods are expensive [S1, unofficial]. The Two-Pass Shadow Casting method by Kajiya and Von Herzen provided a foundational approach for computing shadows in participating media by separating illumination and gathering passes. Later developments, such as Light Propagation Volumes by Kaplanyan, extended these concepts to real-time indirect illumination.

## Sources
[S1] Advanced illumination and view-selection techniques for volume rendering and its application to medical imaging, http://hdl.handle.net/10803/371154, fetched 2026-10-27 (unofficial)
[S2] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
[S3] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S5] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S6] jamestkiernan/webgpu-voxel-raymarching, https://github.com/jamestkiernan/webgpu-voxel-raymarching, fetched 2026-10-24
[S7] [volumetric-clouds-v2][RenderEngine] 1:1 复刻 NadirRoGue/RenderEngine 体积云, https://github.com/ruochenhua/AetherEngine/issues/139, fetched 2026-10-28 (unofficial)
[S8] RaycastVolume.cpp, https://visualizationlibrary.org/documentation/_raycast_volume_8cpp_source.html, fetched 2026-10-27 (unofficial)
