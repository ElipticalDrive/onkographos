# Diffuse Lighting Models

Also called: Diffuse Shading, Lambertian Models, Phong Family

## Overview
Diffuse Lighting Models compute the illumination of surfaces defined by Signed Distance Fields (SDFs) or meshes by estimating surface normals and applying reflectance models to determine how light interacts with the geometry. They solve the problem of shading ray-marched or rasterized geometry without explicit mesh topology (for SDFs) or to smooth faceted appearances (for meshes). This family is distinguished from [Distance Color Blending](791-distance-color-blending.md) by explicitly calculating geometric lighting contributions (diffuse/specular) rather than relying solely on distance-based color mapping, and from [Ray-marched Shadows](1087-ray-marched-shadows.md) by focusing on the local reflectance properties rather than global occlusion calculations.

## Shared principles
All methods in this family operate on a surface point $\mathbf{p}$ and its associated geometric data, primarily the surface normal $\mathbf{n}$ [G]. The normal is typically derived from the Signed Distance Function (SDF) $f(\mathbf{x})$ using the gradient:
$$ \mathbf{n} = \frac{\nabla f(\mathbf{p})}{\|\nabla f(\mathbf{p})\|} $$
where $\nabla f(\mathbf{p})$ is estimated via finite differences, such as solving a linear system based on values at vertices of a tetrahedron, or via back-propagation [S1, unofficial].

The core shading computation generally follows this structure:
1. **Geometric Setup:** Determine the view direction $\mathbf{v}$ (from $\mathbf{p}$ to camera) and light direction $\mathbf{l}$ (from $\mathbf{p}$ to light source) [G].
2. **Occlusion/Ambient Occlusion:** Calculate shadow factors or ambient occlusion values to determine how much light reaches $\mathbf{p}$ from the environment or specific lights [G].
3. **Reflectance Model:** Compute the diffuse and specular components based on the angle between $\mathbf{n}$, $\mathbf{l}$, and the reflection vector $\mathbf{r}$ [G].
4. **Color Accumulation:** Combine the base material color with the lighting contributions, often modulated by depth cueing or atmospheric effects [G].

## Subtopics

### [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md)
Applies the Phong reflection model to SDF surfaces, computing normals via the SDF gradient and accumulating diffuse and specular lighting. It differs from [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by including specular highlights and from [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md) by using the exact reflected light vector rather than the half-vector approximation.

### [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md)
Computes diffuse illumination for SDF surfaces by evaluating the dot product between the surface normal and the light direction. It solves the problem of shading ray-marched geometry without explicit mesh topology. Unlike [Sphere Tracing Basics](52-sphere-tracing-basics.md), which returns a flat base color on hit, this method integrates a light direction and color to produce a shaded result.

### [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md)
Applies the Blinn-Phong reflectance model to SDF-derived normals. It distinguishes itself from [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md) by using the half-vector approximation for specular highlights, which is computationally cheaper, and from [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by including specular and ambient components.

### [Gouraud Shading](4645-gouraud-shading.md)
Renders smooth surfaces by computing lighting colors at mesh vertices and linearly interpolating these intensities across facets. It solves the problem of faceted appearance in polygonal approximations. Unlike Phong shading, which interpolates normals and recomputes lighting per pixel, Gouraud shading performs the lighting calculation only at vertices, making it computationally cheaper but prone to artifacts like missing specular highlights.

### Flat Shading
#### definition
- [S1, unofficial] A shading model assuming uniform lighting where light surrounds an object equally in all directions.
#### code
- [S1, unofficial] glShadeModel(GL_FLAT)
#### pitfall
- [S1, unofficial] Flat shading diminishes the three dimensionality of the scene.

## Background
The concepts of flat and smooth (Gouraud) shading were formalized in early computer graphics APIs, specifically OpenGL, to handle the visual appearance of polygonal meshes [S1, unofficial]. Phong shading, while not explicitly detailed in the provided SDF-specific notes, is a standard extension of these models that interpolates normals rather than colors, providing more accurate specular highlights. The application of these models to Signed Distance Fields (SDFs) is a more recent adaptation, leveraging the analytical or numerical gradients of the SDF to provide normals for ray-marched geometry, effectively bridging the gap between implicit surface rendering and traditional shading models [S1, unofficial][S2, unofficial][S3, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
