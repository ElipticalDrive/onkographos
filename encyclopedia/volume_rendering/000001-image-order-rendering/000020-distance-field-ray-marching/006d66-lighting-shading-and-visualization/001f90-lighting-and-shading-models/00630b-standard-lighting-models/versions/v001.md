# Standard Lighting Models

Also called: Local Illumination Models, Phong-like Shading

## Overview
Standard Lighting Models provide a computationally efficient approximation of surface illumination by combining local lighting components (ambient, diffuse, specular) without simulating global light transport effects like inter-reflections or caustics [G]. They differ from [Distance Color Blending](791-distance-color-blending.md) by relying on geometric normals and reflectance functions rather than volumetric color accumulation, and from [Ray Marching Dither](3269-ray-marching-dither.md) by producing deterministic, non-stochastic results. They serve as the foundational shading layer for [Diffuse Lighting Models](5343-diffuse-lighting-models.md) and [Specular and Reflective Shading](14299-specular-and-reflective-shading.md).

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

### [Diffuse Lighting Models](5343-diffuse-lighting-models.md)
Calculates surface illumination by simulating light scattering, providing the primary visual cue for surface orientation. It excludes specular effects, serving as the base layer for complex shading.

### [Ray Marching with Blinn-Phong Lighting](6898-ray-marching-with-blinn-phong-lighting.md)
Renders implicit surfaces using the Blinn-Phong model. It differs from analytical ray tracing by using iterative distance estimation and from pure SDF rendering by explicitly applying local lighting (diffuse, specular, ambient) rather than just color blending.

### [Specular and Reflective Shading](14299-specular-and-reflective-shading.md)
Handles view-dependent surface reflectance, including specular highlights and mirror-like reflections. It distinguishes itself from [Diffuse Lighting Models](5343-diffuse-lighting-models.md) by explicitly modeling view-dependent reflectance and from [Distance Color Blending](791-distance-color-blending.md) by handling geometric specular bounces.

## Background
Standard lighting models originated in the 1970s and 1980s with the development of rasterization techniques, notably the Phong reflection model (1975) and the Blinn-Phong model (1977) [G]. These models were designed to approximate the appearance of surfaces under local illumination with minimal computational cost, making them suitable for real-time rendering. In the context of distance-field ray marching, they are adapted to work with implicit geometry where normals are derived from the SDF gradient rather than mesh vertices .

