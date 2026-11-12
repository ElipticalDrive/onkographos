# Diffuse Lighting Models
## Overview
Diffuse lighting models compute the illumination of surfaces by estimating how much light reaches a point from light sources, typically based on the angle between the surface normal and the light direction. This family solves the problem of realistic surface appearance in ray-marched and rasterized rendering by providing foundational shading calculations that can be combined with specular models, shadows, and ambient occlusion. It differs from sibling methods like [Ray-marched Shadows](1087-ray-marched-shadows.md) by focusing solely on the reflectance component rather than visibility, and from [Iridescent Material with Perturbation](2327-iridescent-material-with-perturbation.md) by ignoring wavelength-dependent interference effects.
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
Applies the Phong reflection model to SDF surfaces. It differs from [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) by including specular highlights and from [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md) by using the exact reflected light vector rather than the half-vector approximation. Choose when sharp, view-dependent specular highlights are required on SDF geometry.

### [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md)
Computes purely diffuse illumination using the dot product between the normal and light direction. It is the simplest diffuse model, lacking specular components. Choose for matte surfaces or as a base for more complex models.

### [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md)
Applies the Blinn-Phong model to volume intensity gradients. It distinguishes itself by using the half-vector approximation for specular highlights, which is computationally cheaper than Phong's reflection vector. Choose for glossy volume rendering where performance is critical.

### [Gouraud Shading](4645-gouraud-shading.md)
Computes lighting at mesh vertices and interpolates colors across facets. Unlike Phong shading, it does not interpolate normals, leading to less accurate specular highlights but higher performance. Choose for rasterized mesh rendering where smooth color gradients are needed without per-pixel lighting cost.

### Flat Shading
#### definition
- [S1, unofficial] A shading model assuming uniform lighting where light surrounds an object equally in all directions.
#### code
- [S1, unofficial] glShadeModel(GL_FLAT)
#### pitfall
- [S1, unofficial] Flat shading diminishes the three dimensionality of the scene.

## Background
The concepts of flat and smooth (Gouraud) shading originate from early OpenGL specifications, which defined `glShadeModel` to control interpolation behavior [S1, unofficial]. Phong shading, while not explicitly detailed in the provided source, is a standard extension of these principles to include specular reflection. Blinn-Phong was introduced as an optimization to Phong shading. These models are foundational in computer graphics, used in both real-time engines and offline renderers.
## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
