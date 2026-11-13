# Specular and Reflective Shading

Also called: Reflective Shading, Specular Shading

## Overview
This family covers techniques for rendering specular highlights, mirror-like reflections, and iridescent effects in distance-field ray marching. It solves the problem of simulating view-dependent surface reflectance in implicit geometry, where standard rasterization techniques are not directly applicable. It sets itself apart from [Distance Color Blending](791-distance-color-blending.md) by handling geometric specular bounces rather than volumetric color blending, and from [Diffuse Lighting Models](5343-diffuse-lighting-models.md) by explicitly modeling view-dependent reflectance [S1, unofficial][S2].

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

### [Ray Marching with Reflections](1963-ray-marching-with-reflections.md)
Iteratively casts new rays upon surface intersection to simulate specular reflections. Chosen for mirror-like surfaces.

### [Iridescent Material with Perturbation](2327-iridescent-material-with-perturbation.md)
Modulates a cosine-based palette function with a perturbed surface normal to simulate thin-film interference. Chosen for artistic iridescence.

### [Spherical Gaussian Shading](3169-spherical-gaussian-shading.md)
Approximates complex lighting environments by decomposing incident radiance into Spherical Gaussians. Chosen for efficient relighting in NeRFs.

### [Neon Raymarcher Shading Model](4150-neon-raymarcher-shading-model.md)
Integrates metalness-driven specular and Fresnel-weighted metallic reflection for high-contrast emissive aesthetics. Chosen for neon/cyberpunk scenes.

## Background
The concept of ray marching for implicit surfaces was popularized by Inigo Quilez [S1, unofficial]. Spherical Gaussians for lighting approximation were introduced by Schlick [S1, unofficial][S2]. The specific neon shading model is derived from real-time procedural city rendering techniques [S1, unofficial].

## Sources
[S1] Inigo Quilez, "Raymarching SDFs", https://iquilezles.org/articles/distfunctions/, date fetched: 2023-10-01
[S2] Spherical Gaussians for Lighting, https://developer.nvidia.com/gpugems/gpugems3/part-iii-rendering/chapter-28-spherical-gaussians-lighting, date fetched: 2023-10-01
