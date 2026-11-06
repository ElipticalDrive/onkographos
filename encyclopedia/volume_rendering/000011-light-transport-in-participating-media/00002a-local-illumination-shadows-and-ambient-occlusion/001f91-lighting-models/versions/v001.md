# Lighting Models

Also called: Lighting Calculation

## Overview
Lighting Models in volume rendering apply local illumination heuristics to scalar field data to enhance the perception of 3D structure and surface orientation [S1, unofficial]. Unlike physically based light transport methods that solve the radiative transfer equation, these models modulate existing volume samples or pixel colors using diffuse and specular terms derived from the data gradient [S2, unofficial]. This family distinguishes itself from siblings like [Two-Pass Shadow Casting (Kajiya and Von Herzen)](2196-two-pass-shadow-casting-kajiya-and-von-herzen.md) by focusing on local shading rather than global shadow computation, and from [Path Tracing and Sampling](2526-path-tracing-and-sampling.md) by using deterministic or semi-analytical approximations instead of stochastic integration.

## Shared principles
The core mechanism involves computing a surface normal approximation from the volume data gradient $\mathbf{g} = \nabla V$ and applying a local illumination model (e.g., Phong, Lambertian, or microfacet-based) to determine the color contribution of a light source.

1.  **Gradient Estimation:** The normal $\mathbf{n}$ is approximated by normalizing the gradient vector $\mathbf{g}$ at the intersection point or voxel center. Gradients are typically computed via the derivative of the interpolation function (e.g., trilinear) or finite differences [S1, unofficial].
2.  **Illumination Calculation:** The lighting contribution $C_{shade}$ is calculated using the dot product of the normal $\mathbf{n}$, the light direction $\mathbf{l}$, and potentially the view direction $\mathbf{v}$ or half-vector $\mathbf{h}$:
    $$ C_{shade} = k_d (\mathbf{n} \cdot \mathbf{l}) C_{diffuse} + k_s (\mathbf{n} \cdot \mathbf{h})^\alpha C_{specular} $$
    where $k_d, k_s$ are diffuse/specular weights, $\alpha$ is shininess, and $C_{diffuse}, C_{specular}$ are material/light colors [S1, unofficial].
3.  **Blending:** The shading contribution is blended with the underlying volume color $C_{vol}$ (from ray casting or slicing) using an opacity factor $\beta$:
    $$ C_{final} = C_{vol} + \beta C_{shade} $$
    [S1, unofficial].

## Subtopics

### [Shading models](837-shading-models.md)
Uses a simplified Phong-like heuristic on data gradients to modulate volume colors. Chosen for interactive scalar field visualization where physical accuracy is secondary to structural clarity.

### [Shading for Semi-Transparent Volumes](2144-shading-for-semi-transparent-volumes.md)
Enables local diffuse illumination for semi-transparent volumes using multi-stage rasterization and register combiners. Chosen for early PC hardware constraints where 3D texture interpolation was unavailable.

### [Cook-Torrance BRDF](2509-cook-torrance-brdf.md)
A physically based model for opaque surfaces using microfacet theory (GGX distribution, Schlick Fresnel, Schlick-GGX geometry). Chosen for realistic rendering of metallic and dielectric surfaces.

### [Direct Lighting and Shading](3953-direct-lighting-and-shading.md)
Computes direct illumination with a non-standard GGX-like specular term ($H = \text{normalize}(L - V)$) and exponential tone mapping. Chosen for voxel-based/splat-based pipelines requiring immediate shading.

### [Whitted-Style Shading with Diffuse/Glossy Materials](4896-whitted-style-shading-with-diffuse-glossy-materials.md)
Combines ambient, diffuse, and glossy reflection with shadow ray casting. Chosen for ray-traced scenes with limited recursion depth and analytical specular approximations.

### Two-sided Lighting
A variant where directional lights illuminate both front and back faces.

### Two-sided Lighting
Under two-sided lighting, each directional light shines in two directions: both the specified direction and the opposite direction [S3]. This is enabled via the `VP_LIGHT_BOTH_SIDES` option in the VolPack API [S3]. The light color is set using `vpSetLight` with `VP_COLOR` (RGB components 0.0-1.0) and the direction using `VP_DIRECTION` (transformed by the current modeling matrix) [S3]. Light 0 is enabled by default; others require explicit `vpEnable` calls [S3].

## Background
The concept of shading volume data using gradients originated in the early days of volume visualization to provide depth cues [S1, unofficial]. The VolPack library (Stanford) provided early API support for directional lighting and two-sided shading [S3]. The integration of Phong and Blinn-Phong models into volume ray casting became standard for interactive applications [S1, unofficial]. Later, physically based models like Cook-Torrance were adapted for volume rendering to handle complex material properties like metals and rough surfaces [S1, unofficial]. The use of hardware combiners for semi-transparent shading emerged as a solution to the lack of 3D texture interpolation support in early GPUs [S1, S2, unofficial].

## Sources
[S1] rokcej/volumetric-ray-casting, https://github.com/rokcej/volumetric-ray-casting, fetched 2026-10-26 (unofficial)
[S2] Sample - Ray Marching and Signed Distance Functions, https://github.com/ComputeNode/cyfra/pull/13, fetched 2026-10-27 (unofficial)
[S3] VolPack User's Guide, http://www-graphics.stanford.edu/software/volpack/vp_userguide.html, fetched 2026-10-28
