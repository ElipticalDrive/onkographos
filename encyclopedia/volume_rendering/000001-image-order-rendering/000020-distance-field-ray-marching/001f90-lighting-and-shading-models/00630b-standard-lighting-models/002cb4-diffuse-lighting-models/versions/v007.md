# Diffuse Lighting Models

Also called: Diffuse Shading, Lambertian Shading, Phong Diffuse, Blinn-Phong Diffuse

## Overview
Diffuse lighting models calculate the illumination of surfaces by simulating the scattering of light, providing the primary visual cue for surface orientation and shape in ray-marched and rasterized rendering. This family distinguishes itself from sibling methods like [Ray Marching with Reflections](1963-ray-marching-with-reflections.md) or [Iridescent Material with Perturbation](2327-iridescent-material-with-perturbation.md) by focusing exclusively on non-specular, non-directional reflectance, often serving as the base layer for more complex shading [G].

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
Applies the Phong reflection model to SDF surfaces, distinguishing itself from [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by including specular highlights and from [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md) by using the exact reflected light vector rather than the half-vector approximation [G].

### [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md)
Calculates pure diffuse illumination by clamping the dot product of the normal and light direction to zero, focusing solely on the diffuse component unlike [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md) or [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md) which include specular highlights.

### [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md)
Applies the Blinn-Phong model to volume data, using the half-vector approximation for specular highlights which is computationally cheaper than calculating the reflection vector, and differs from [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by including specular and ambient components.

### [Gouraud Shading](4645-gouraud-shading.md)
Computes lighting colors at mesh vertices and linearly interpolates them across facets, solving the faceted appearance of polygonal approximations by creating continuous color gradients, unlike Phong shading which interpolates normals.

### Flat Shading
#### definition
- [S1, unofficial] A shading model assuming uniform lighting where light surrounds an object equally in all directions.
#### code
- [S1, unofficial] glShadeModel(GL_FLAT)
#### pitfall
- [S1, unofficial] Flat shading diminishes the three dimensionality of the scene.

## Background
The concept of flat shading was formalized in early graphics APIs like OpenGL, which provided `glShadeModel(GL_FLAT)` to specify uniform lighting across primitives [S1, unofficial]. Gouraud shading, also known as smooth shading, was introduced to provide subtle color changes across surfaces by interpolating vertex colors, assuming light comes from a particular direction [S1, unofficial]. These models are foundational in visualizing data, such as in Fire Dynamics Simulation tools like Smokeview, where they help convey the three-dimensionality of objects [S1, unofficial].

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
