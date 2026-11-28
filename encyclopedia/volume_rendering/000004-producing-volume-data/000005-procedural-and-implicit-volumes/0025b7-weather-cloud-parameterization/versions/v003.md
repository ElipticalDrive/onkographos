# Weather & Cloud Parameterization

Also called: atmospheric settings, cloud simulation parameters

## Overview
Weather & Cloud Parameterization defines the global physical and visual configuration for volumetric cloud rendering, controlling parameters such as coverage, density, cloud type, geometry bounds, lighting angles, and noise interaction weights [S1, S4, unofficial]. It solves the problem of decoupling the high-level atmospheric state from the low-level procedural noise generation and raymarching integration, allowing a single configuration to drive multiple rendering passes or simulation variants [S1, unofficial]. It differs from its siblings by providing the *input data structures and global constants* rather than the algorithmic implementation of density field generation ([Procedural Volume Cloud Modeling](2256-procedural-volume-cloud-modeling.md)) or light integration ([Volumetric Cloud Raymarching](1248-volumetric-cloud-raymarching.md)).

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering. The core mechanism involves:
1.  **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [S1, S3, unofficial].
2.  **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [S1, S12].
3.  **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [S1, S3, unofficial].

## Subtopics

### [Volumetric Cloud Raymarching](1248-volumetric-cloud-raymarching.md)
Renders cumulus and stratiform clouds by tracing rays through a 3D procedural density field, integrating light attenuation via Beer's Law and scattering using phase functions. Unlike solid geometry raymarching which terminates on hit, this method accumulates color and transmittance over fixed or adaptive steps, allowing for soft self-shadowing and god rays. It differs from its parent by implementing the actual radiative transfer integration, and from [Procedural Volume Cloud Modeling](2256-procedural-volume-cloud-modeling.md) by focusing on the real-time fragment shader traversal rather than just the noise generation.

### [Procedural Volume Cloud Modeling](2256-procedural-volume-cloud-modeling.md)
Constructs the volumetric density field of clouds by evaluating mathematical functions and sampling noise textures at arbitrary 3D coordinates, rather than relying on pre-baked voxel grids or static meshes. It solves the problem of generating infinite-resolution, seamless, and physically plausible cloud structures that can be queried during raymarching. It differs from [Volumetric Cloud Raymarching](1248-volumetric-cloud-raymarching.md) by focusing on the *generation* of the scalar field $f(\mathbf{x})$ rather than the integration of light through it, and from [Atmospheric Height Gradient](4813-atmospheric-height-gradient.md) by incorporating spatially varying weather maps and multi-octave noise erosion rather than a simple vertical falloff.

### [Atmospheric Height Gradient](4813-atmospheric-height-gradient.md)
Defines the vertical density distribution profile for volumetric cloud layers based on cloud type (Stratus, Cumulus, Cumulonimbus, Cirrus). It computes a scalar shape factor representing the relative density at a given altitude within a cloud layer, creating bell-shaped or plateau-shaped distributions. It differs from [Volumetric Cloud Raymarching](1248-volumetric-cloud-raymarching.md) by providing the underlying scalar field definition rather than the integration technique, and from [Procedural Volume Cloud Modeling](2256-procedural-volume-cloud-modeling.md) by focusing specifically on the vertical stratification logic rather than horizontal noise composition.

### Cloud Coverage Remapping
A minor variant of density composition where the base cloud shape is remapped based on a global coverage parameter before final scaling [S2, unofficial]. The process reads the cloud coverage value (typically from the red channel of a weather map, default 0.6) and applies a clamped remapping function to the base cloud density: `base_cloud_with_coverage = remapClampedBeforeAndAfter(baseCloud, cloud_coverage, 1.0, 0.0, 1.0)` [S2, unofficial]. This reduces the base cloud density as coverage increases, preventing over-saturation. The result is then multiplied by the cloud coverage value to ensure the final aesthetic density increases with the coverage setting: `base_cloud_with_coverage *= cloud_coverage` [S2, unofficial].

### Physical Shading
A minor variant of the scattering calculation that explicitly models multiple scattering octaves and per-Gaussian shading properties [S3, unofficial]. The sun direction $\mathbf{v}_l$ is obtained per-frame from the dataset but can be replaced arbitrarily during inference to enable relighting [S3, unofficial]. The radiance $L$ is calculated as:
$$ L = \boldsymbol{\omega} \mathbf{L}_{\text{light}} \sum_n w_n \mathbf{T}_{\text{light}}^{b^n} \mathbf{P}(\mathbf{v}_i, \mathbf{v}_o; g \mathbf{c}^n) $$
Where $\boldsymbol{\omega}$ is the scattering albedo (RGB), $\mathbf{L}_{\text{light}}$ is the light source intensity, $w_n$ are the multiple scattering octave weights, $\mathbf{T}_{\text{light}}$ is the self-shadow transmittance, $b^n$ is the exponent for the $n$-th octave, $\mathbf{P}$ is the Henyey-Greenstein phase function, $\mathbf{v}_i$ is the incident direction ($-\mathbf{v}_l$), $\mathbf{v}_o$ is the view direction, $g$ is the anisotropy factor, and $\mathbf{c}^n$ are octave-specific coefficients [S3, unofficial].

### Cloud Configuration Parameters
A specific data structure definition used in Rust/wgpu-based render engines (e.g., AetherEngine) to pass cloud settings to compute shaders [S4, unofficial]. The `CloudConfig` struct includes: `cloud_type` (f32: 0=stratus, 0.5=stratocumulus, 1=cumulus), `coverage_multiplier` (f32), `wind_direction` ([f32; 3]), `wind_speed` (f32), `cloud_color` ([f32; 3]), `inner_sphere_radius` (f32), `outer_sphere_radius` (f32), `sphere_y_offset` (f32), `max_render_distance` (f32), `cloud_top_offset` (f32), `weather_texture_scale` (f32), `base_noise_scale` (f32), `high_frequency_noise_scale` (f32), `high_frequency_noise_uv_scale` (f32), `high_frequency_noise_h_scale` (f32), and `quality` (enum: Low, Medium, High) [S4, unofficial]. All fields correspond directly to RenderEngine Settings parameters [S4, unofficial].

## Background
The concept of parameterizing weather for volumetric rendering originated in real-time graphics engines seeking to balance physical accuracy with performance. Early implementations relied on static height fields, but modern approaches integrate dynamic weather maps and multi-octave noise erosion [S1, S2, unofficial]. The specific parameter structures described in [S1, unofficial] and [S4, unofficial] reflect the evolution from simple shader uniforms to complex, serialized configuration files (e.g., RON) that drive both procedural generation and raymarching passes [S1, S4, unofficial]. The use of Henyey-Greenstein phase functions and Beer's Law variants has become standard for approximating light transport in participating media within these parameterized frameworks [S3, S4, unofficial].

## Sources
[S1] harkawataryan/Volumetric-Cloud-Renderer, https://github.com/harkawataryan/Volumetric-Cloud-Renderer, fetched 2026-10-27 (unofficial)
[S2] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
[S3] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
[S4] [volumetric-clouds-v2][RenderEngine] 1:1 复刻 NadirRoGue/RenderEngine 体积云, https://github.com/ruochenhua/AetherEngine/issues/139, fetched 2026-10-28 (unofficial)
