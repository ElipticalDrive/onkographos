# Diffuse Lighting Models
## Overview
Diffuse lighting models calculate surface illumination by simulating light scattering, providing the primary visual cue for surface orientation and shape in ray-marched and rasterized rendering [G]. This family distinguishes itself from sibling methods like [Distance Color Blending](791-distance-color-blending.md) and [Specular and Reflective Shading](14299-specular-and-reflective-shading.md) by focusing exclusively on non-specular reflectance, often serving as the base layer for more complex shading [G].

## Shared principles
All methods in this family operate on a surface point $\mathbf{p}$ and its associated geometric data, primarily the surface normal $\mathbf{n}$, as detailed in [Lighting and Shading Models](2055-lighting-and-shading-models.md) [G].

## Subtopics

### [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md)
Applies the Phong reflection model to SDF surfaces, distinguishing itself from [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by including specular highlights and from [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md) by using the exact reflected light vector rather than the half-vector approximation. Choose it when a shinier appearance with precise specular reflections is required.

### [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md)
Calculates pure diffuse illumination by clamping the dot product of the normal and light direction to zero, focusing solely on the diffuse component unlike [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md) or [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md) which include specular highlights. Choose it for physically plausible, matte shading that reveals surface curvature.

### [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md)
Applies the Blinn-Phong model to volume data, using the half-vector approximation for specular highlights which is computationally cheaper than calculating the reflection vector, and differs from [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by including specular and ambient components. Choose it for efficient glossy surface simulation in volume rendering.

### [Gouraud Shading](4645-gouraud-shading.md)
Computes lighting colors at mesh vertices and linearly interpolates them across facets, solving the faceted appearance of polygonal approximations by creating continuous color gradients, unlike Phong shading which interpolates normals. Choose it for performance-critical rasterized rendering of curved surfaces.

#### Flat Shading

### definition
- [S1, unofficial] A shading model assuming uniform lighting where light surrounds an object equally in all directions.

### code
- [S1, unofficial] glShadeModel(GL_FLAT)

### pitfall
- [S1, unofficial] Flat shading diminishes the three dimensionality of the scene.

## Background
The concept of flat shading was formalized in early graphics APIs like OpenGL, which provided `glShadeModel(GL_FLAT)` to specify uniform lighting across primitives [S1, unofficial]. Gouraud shading, also known as smooth shading, was introduced to provide subtle color changes across surfaces by interpolating vertex colors, assuming light comes from a particular direction [S1, unofficial]. These models are foundational in visualizing data, such as in Fire Dynamics Simulation tools like Smokeview [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
