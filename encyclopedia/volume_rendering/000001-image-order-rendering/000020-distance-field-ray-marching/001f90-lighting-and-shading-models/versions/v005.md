# Lighting and Shading Models


## Overview
Lighting and Shading Models compute the color of surface points identified by distance-field ray marching by estimating surface normals and applying reflectance models to determine how light interacts with the geometry [G]. This family solves the problem of shading ray-marched geometry without explicit mesh topology, distinguishing itself from [Distance Color Blending](791-distance-color-blending.md) by calculating geometric lighting contributions (diffuse/specular) rather than relying solely on distance-based color mapping, and from [Ray-marched Shadows](1087-ray-marched-shadows.md) by focusing on local reflectance properties rather than global occlusion calculations [G].

## Shared principles
All methods in this family operate on a surface point $\mathbf{p}$ and its associated geometric data, primarily the surface normal $\mathbf{n}$ [G]. The normal is typically derived from the Signed Distance Function (SDF) $f(\mathbf{x})$ using the gradient:
$$ \mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|} $$
where $\nabla f(\mathbf{p})$ is estimated via finite differences, such as solving a linear system based on values at vertices of a tetrahedron, or via back-propagation [G].

The core shading computation generally follows this structure:
1. **Geometric Setup:** Determine the view direction $\mathbf{v}$ (from $\mathbf{p}$ to camera) and light direction $\mathbf{l}$ (from $\mathbf{p}$ to light source) [G].
2. **Occlusion/Ambient Occlusion:** Calculate shadow factors or ambient occlusion values to determine how much light reaches $\mathbf{p}$ from the environment or specific lights [G].
3. **Reflectance Model:** Compute the diffuse and specular components based on the angle between $\mathbf{n}$, $\mathbf{l}$, and the reflection vector $\mathbf{r}$ [G].
4. **Color Accumulation:** Combine the base material color with the lighting contributions, often modulated by depth cueing or atmospheric effects [G].

## Subtopics

### [Distance Color Blending](791-distance-color-blending.md)
Enhances depth perception by blending sampled voxel colors with a background color based on distance from the viewer. It solves the foreground/background distinction problem in dense volumetric data by applying a non-linear transparency falloff that dims distant objects and shifts their hue, simulating atmospheric haze or aerial perspective. Unlike standard lighting models that compute surface reflectance, this method operates on the accumulated color along the ray, modulating it with depth. Choose it when foreground/background distinction is needed in dense volumetric data without explicit mesh topology.

### [Ray-marched Shadows](1087-ray-marched-shadows.md)
Determines if a surface point is occluded from a light source by performing a sphere trace from the point towards the light. Unlike rasterization-based shadow maps, this method computes occlusion directly from the Signed Distance Function (SDF) of the scene, producing alias-free hard shadows with a single additional traversal per light. It sets itself apart from [Soft Shadow Ray Marching](1107-soft-shadow-ray-marching.md) by returning a binary occlusion result rather than a continuous penumbra factor, and from [Shadow Casting Algorithm](1657-shadow-casting-algorithm.md) by marching from surface points toward lights rather than projecting light through a volumetric dataset. Choose it for alias-free hard shadows directly from SDFs, avoiding rasterization shadow maps.

### [Ambient Occlusion via SDF](1631-ambient-occlusion-via-sdf.md)
Approximates the darkening of crevices and contact points by sampling the signed distance function (SDF) along the surface normal. It detects occlusion by comparing the actual distance to nearby geometry against the expected distance if the surface were isolated; a smaller actual distance indicates an obstacle is closer, reducing the ambient light contribution. Unlike screen-space methods or shadow maps, this technique requires no separate rendering pass, relying solely on the scene's distance function sampled differently. It is computationally cheaper than full ray-marched shadows but provides a local approximation of global illumination effects. Choose it for computationally cheaper local approximation of global illumination without separate rendering passes.

### [Ray Marching with Reflections](1963-ray-marching-with-reflections.md)
Extends standard distance-field ray marching by iteratively casting new rays upon surface intersection to simulate specular reflections. It solves the problem of rendering mirror-like or glossy surfaces in implicit geometry by continuing the ray path after a hit, rather than terminating at the first surface. It sets itself apart from [Distance Color Blending](791-distance-color-blending.md) by handling geometric specular bounces rather than volumetric color blending, and from [Screen-Space Reflections](9698-screen-space-reflections.md) by operating in 3D space via Signed Distance Functions (SDFs) rather than 2D screen-space depth buffers. Choose it for mirror-like or glossy surfaces in implicit geometry, operating in 3D space rather than screen-space.

### [Iridescent Material with Perturbation](2327-iridescent-material-with-perturbation.md)
Renders iridescent surfaces by modulating a cosine-based palette function with a perturbed surface normal, simulating thin-film interference colors, while combining this base color with multi-component specular highlights and directional shadowing. It differs from standard diffuse or simple specular models by generating color variation through geometric perturbation rather than wavelength-dependent physics, and it differs from physically accurate thin-film models by using a computationally cheaper, artistic approximation driven by position and normal noise. Choose it for artistic thin-film interference simulation via geometric perturbation rather than wavelength-dependent physics.

### [Multi-Layer Shading / OmniShader](2511-multi-layer-shading-omnishader.md)
Computes eight distinct visualization layers simultaneously in a single forward pass, returning them as a tuple of tensors. It sets itself apart from sibling methods like [Distance Color Blending](791-distance-color-blending.md) or [Ray-marched Shadows](1087-ray-marched-shadows.md) by decoupling geometric, photometric, and stylistic attributes into parallel outputs rather than combining them into a single RGB image, allowing for flexible post-processing or neural network ingestion. Choose it when decoupling geometric, photometric, and stylistic attributes into parallel outputs is required for post-processing or neural ingestion.

### [Spherical Gaussian Shading](3169-spherical-gaussian-shading.md)
Approximates complex lighting environments and specular reflections by decomposing incident radiance into a sum of Spherical Gaussians, which are then rendered as discrete point lights or integrated into a microfacet BRDF model. It solves the problem of efficient relighting in scenes with spatially variant lighting, such as Neural Radiance Fields (NeRFs), by avoiding the high computational cost of full spherical harmonic integration. Unlike [Diffuse Lighting Models](5343-diffuse-lighting-models.md) which ignore view-dependent effects, SG Shading explicitly models specular highlights and view-dependent reflectance. Choose it for efficient relighting in scenes with spatially variant lighting like NeRFs, avoiding full spherical harmonic integration.

### [Ray Marching Dither](3269-ray-marching-dither.md)
A family of techniques used in distance-field ray marching to suppress banding and layering artifacts caused by low step counts or discrete sampling intervals. Unlike standard fixed-step marching which produces visible intensity bands, these methods introduce controlled randomness to the sampling process—either by perturbing the ray's UV coordinates based on hit distance, offsetting the initial depth using a blue noise texture, or jittering the step size itself. This breaks up coherent aliasing patterns while maintaining the sharpness of volumetric features like god rays. Choose it to break up coherent aliasing patterns from low step counts while maintaining sharp volumetric features.

### [Custom Shading Models](3660-custom-shading-models.md)
Provide a mechanism for users to define arbitrary illumination logic for volume rendering, bypassing standard predefined lighting equations. They solve the problem of rigid lighting constraints by allowing per-voxel or per-surface-point computation of color based on user-defined material properties, geometric data, and environmental factors. What sets this family apart from its siblings is the architectural approach to integration: some methods inject custom logic via API callbacks into a rendering pipeline, while others implement self-contained mathematical models or procedural rules directly within the shading step. Choose it when rigid lighting constraints must be bypassed via API callbacks or self-contained mathematical models.

### [Neon Raymarcher Shading Model](4150-neon-raymarcher-shading-model.md)
A real-time shading technique for procedural neon city scenes that computes diffuse, specular, ambient occlusion, metallic reflection, and emissive contributions per pixel. It distinguishes itself by integrating a metalness-driven specular model and a Fresnel-weighted metallic reflection term tailored for high-contrast, emissive aesthetics. Choose it for high-contrast, emissive aesthetics with metalness-driven specular and Fresnel-weighted reflections.

### [Diffuse Lighting Models](5343-diffuse-lighting-models.md)
Calculate the illumination of surfaces by simulating the scattering of light, providing the primary visual cue for surface orientation and shape in ray-marched and rasterized rendering. This family distinguishes itself from sibling methods like [Ray Marching with Reflections](1963-ray-marching-with-reflections.md) or [Iridescent Material with Perturbation](2327-iridescent-material-with-perturbation.md) by focusing exclusively on non-specular, non-directional reflectance, often serving as the base layer for more complex shading. Choose it as the base layer for non-specular, non-directional reflectance in ray-marched or rasterized rendering.

### [Ray Marching with Blinn-Phong Lighting](6898-ray-marching-with-blinn-phong-lighting.md)
Renders scenes defined by implicit surfaces (Signed Distance Fields) by casting rays from a camera, iteratively marching them to a surface, and shading the hit point using the Blinn-Phong reflectance model. It differs from analytical ray tracing by using iterative distance estimation to handle complex, non-analytic geometry, and it differs from pure SDF rendering by applying a specific local lighting approximation (Blinn-Phong) rather than global illumination or complex material models. Choose it for applying a specific local lighting approximation (Blinn-Phong) to complex, non-analytic geometry via iterative distance estimation.

### Material Evaluation Strategy
Write a material ID, the distance, and the local coordinate `p` into global variables whenever an object's distance is smaller than the stored distance. At the end of the rendering process, evaluate the material to get color, roughness, etc., and perform shading [S1, unofficial].

### Local Distance Field Ambient Occlusion
A modern approach offloading meshing and AO to compute shaders to achieve smooth, high-fidelity AO that avoids the rigid look of per-vertex discrete levels [S2, unofficial]. Voxel chunk data is held in a 3D texture, and a compute shader evaluates the local density field within a small spherical kernel (e.g., a $3 \times 3 \times 3$ or $5 \times 5 \times 5$ window) around each surface point. If the procedural generation pipeline generates a distance field rather than raw block binaries, short-range cone tracing through the local SDF provides smooth, realistic contact shadows baked directly into the output mesh buffers during the surface extraction pass (such as Marching Cubes or Surface Nets) [S2, unofficial].

## Background
Not given in the sources.

## Sources
[S1] yeataro/TD-Raymarching-System, https://github.com/yeataro/TD-Raymarching-System, fetched 2026-10-27 (unofficial)
[S2] Baking Dynamic Ambient Occlusion in Procedural Voxels, https://salivity.github.io/game-development/article/baking-dynamic-ambient-occlusion-in-procedural-voxels, fetched 2026-10-29 (unofficial)
