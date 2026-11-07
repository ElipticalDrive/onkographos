# Diffuse Lighting Models

Also called: Diffuse Shading, Lambertian Shading, Phong Shading, Blinn-Phong Shading, Gouraud Shading

## Overview
Diffuse Lighting Models compute the illumination of surfaces defined by Signed Distance Fields (SDFs) or volume data by estimating surface normals and applying reflectance models. They solve the problem of rendering flat, unshaded ray-marched geometry as three-dimensional objects with realistic lighting. This family is distinguished from [Distance Color Blending](791-distance-color-blending.md) by incorporating directional light vectors and surface orientation, and from [Ray-marched Shadows](1087-ray-marched-shadows.md) by focusing on local reflectance rather than global occlusion, though they often serve as the base for shadow calculations.

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
Applies the Phong reflection model to SDF surfaces. It computes exact specular highlights using the reflected light vector $\mathbf{r}$, distinguishing it from [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md) which lacks speculars, and from [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md) which uses the half-vector approximation. Choose this when high-quality specular highlights are required for SDF geometry.

### [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md)
Computes purely diffuse illumination using the dot product between the surface normal and light direction. It solves the problem of shading ray-marched geometry without explicit mesh topology. Unlike [Sphere Tracing Basics](52-sphere-tracing-basics.md) which returns flat colors, this integrates light direction. Choose this for matte, non-shiny materials.

### [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md)
Applies the Blinn-Phong model to volume data during ray marching. It distinguishes itself from unshaded methods by calculating normals via volume gradients and combining diffuse, specular, and ambient components. It differs from Phong shading by using the half-vector for specular highlights, which is computationally cheaper and avoids the reflection vector calculation. Choose this for volumetric data where performance is a concern or for a softer specular appearance.

### [Gouraud Shading](4645-gouraud-shading.md)
A vertex-based technique that computes lighting at mesh vertices and linearly interpolates the final color across primitives. It differs from sibling methods by interpolating color rather than normals; [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md) interpolates normals to re-evaluate the lighting model per-pixel. Choose this for rasterized mesh rendering rather than ray-marched SDFs, or when performance is critical and per-pixel lighting is not required.

### Flat Shading
#### definition
A shading model assuming uniform lighting where light surrounds an object equally in all directions [S1, unofficial].
#### code
`glShadeModel(GL_FLAT)` [S1, unofficial]
#### pitfall
Flat shading diminishes the three dimensionality of the scene [S1, unofficial].

## Background
The concepts of flat and smooth (Gouraud) shading were formalized in early computer graphics APIs, such as OpenGL, to define how lighting is applied to primitives [S1, unofficial]. Phong shading, introduced by Bui Tuong Phong, provided a more accurate model for specular highlights by interpolating normals. Blinn-Phong, proposed by James F. Blinn, offered an approximation that became standard in many real-time rendering contexts. In the context of volume rendering and SDFs, these models have been adapted to work with implicit surfaces defined by distance fields, where normals are derived from the field's gradient rather than mesh topology.

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
