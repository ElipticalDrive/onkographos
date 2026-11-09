# Lighting Models

Also called: Lighting Calculation

## Overview
Lighting models evaluate local illumination at a point $\mathbf{p}$ within or on the boundary of a volume to enhance the perception of 3D shape, surface detail, and orientation [S1, unofficial]. Unlike full light transport methods that simulate scattering, these models approximate surface-like appearance using local shading (diffuse/specular) and shadowing [S1, unofficial]. They are distinguished from participating media scattering methods (e.g., [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md)) by treating the volume as a collection of pseudo-surfaces, and from frequency-domain approximations (e.g., [Frequency Domain Directional Shading](852-frequency-domain-directional-shading.md)) by operating in spatial domain [S1, unofficial].

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

### [Shading models](837-shading-models.md)
Applies local illumination to scalar field data to enhance 3D shape perception. It treats the volume as pseudo-surfaces, computing diffuse and specular contributions based on data values and gradients. It distinguishes itself from [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md) by managing opacity and light transport modes in a single pass, and from [Lambert Shading with Shadow Rays](1133-lambert-shading-with-shadow-rays.md) by including specular highlights.

### [Shading for Semi-Transparent Volumes](2144-shading-for-semi-transparent-volumes.md)
Enables local diffuse illumination for semi-transparent volumes using multi-stage rasterization hardware (register combiners) for single-pass interactive performance. It differs from isosurface shading by using alpha blending and from 3D texture methods by sacrificing trilinear interpolation to fit gradient and color data into separate 2D textures.

### [Cook-Torrance BRDF](2509-cook-torrance-brdf.md)
A physically based rendering (PBR) model using the microfacet surface model to represent real-world reflectance, distinguishing metals and dielectrics. It ensures energy conservation and models roughness via statistical microfacet alignment, separating diffuse and specular lobes with energy-conserving weights and Fresnel effects.

### [Direct Lighting and Shading](3953-direct-lighting-and-shading.md)
Evaluates illumination from discrete light sources and ambient terms, approximating surface appearance through local shading. It differs from [Shading models](837-shading-models.md) by integrating direct light visibility into volume traversal, and from [Cook-Torrance BRDF](2509-cook-torrance-brdf.md) by often employing simplified specular approximations.

### [Whitted-Style Shading with Diffuse/Glossy Materials](4896-whitted-style-shading-with-diffuse-glossy-materials.md)
Evaluates local illumination by combining ambient lighting with direct light contributions and shadow rays. It limits recursion to direct lighting and simple reflections, treating interaction as a local surface-like reflection model. Chosen for real-time performance with opaque/semi-transparent surfaces requiring approximate shadows and basic specular highlights.

### Two-sided Lighting
Under two-sided lighting, each directional light shines in two directions: both the specified direction and the opposite direction [S3]. Implementation involves setting light color and direction via `vpSetLight` and enabling the light with `vpEnable`, specifically using the `VP_LIGHT_BOTH_SIDES` option [S3].

### Atmospheric Lighting Model
A lighting model that attenuates light through the volume, providing simple but robust lighting without relying on a surface normal [S4, unofficial]. It is used for multi-field gradients because the gradient orientation may not correspond to the surface of the classified region and can flip direction [S4, unofficial].

### Spot Light with Cone Falloff
A spot light implementation with smooth cone falloff and optional IES profile, acting as a delta light [S5, unofficial]. It involves implementing smooth cone falloff and blending with optional IES profile [S5, unofficial].

## Background
Lighting models for volume rendering originated with the need to enhance the visual interpretation of scalar field data. Early implementations focused on simple diffuse and specular shading applied to ray-casted volumes [S1, unofficial]. The use of hardware acceleration, such as register combiners, allowed for interactive shading of semi-transparent volumes on early PC hardware by offloading dot product calculations [S1, unofficial][S2, unofficial]. Physically based models like Cook-Torrance were adapted for volumetric contexts to improve material realism [S1, unofficial][S5, unofficial]. Atmospheric lighting models were developed specifically for multivariate data visualization where gradient-based normals are unreliable [S4, unofficial].

## Sources
[S1] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S2] Sample - Ray Marching and Signed Distance Functions, https://github.com/ComputeNode/cyfra/pull/13, fetched 2026-10-27 (unofficial)
[S3] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
[S4] Volume rendering multivariate data to visualize meteorological ..., https://www.researchgate.net/publication/262237652_Volume_rendering_multivariate_data_to_visualize_meteorological_simulations_a_case_study, fetched 2026-10-29 (unofficial)
[S5] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
