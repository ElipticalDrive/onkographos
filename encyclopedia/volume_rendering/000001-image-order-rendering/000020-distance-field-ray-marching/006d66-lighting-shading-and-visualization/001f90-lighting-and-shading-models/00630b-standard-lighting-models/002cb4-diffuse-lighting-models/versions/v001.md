# Diffuse Lighting Models

Also called: Diffuse Shading, Lambertian Shading, Phong Diffuse

## Overview
Diffuse Lighting Models calculate the scattering of light from a surface, providing the primary cue for shape and depth in ray-marched volume rendering and SDF rendering. Unlike specular models that highlight surface gloss or ambient models that provide uniform illumination, diffuse models compute light intensity based on the angle between the surface normal and the light direction. This family is distinct from [Distance Color Blending](791-distance-color-blending.md) by integrating directional light vectors and from [Ray-marched Shadows](1087-ray-marched-shadows.md) by assuming unobstructed light paths unless combined with shadowing techniques.

## Shared principles
All models in this family rely on the geometric relationship between the surface normal $\mathbf{N}$ and the light direction $\mathbf{L}$. The core mechanism involves computing the dot product $\mathbf{N} \cdot \mathbf{L}$ to determine the projected area of the light source onto the surface. This value is clamped to $[0, 1]$ to prevent back-face illumination and negative lighting contributions. The final color contribution is typically calculated as:
$$ C_{diffuse} = C_{base} \times (\mathbf{N} \cdot \mathbf{L})_+ \times C_{light} $$
where $(\cdot)_+$ denotes the positive part (clamping negatives to zero), $C_{base}$ is the material albedo, and $C_{light}$ is the light color intensity. Normals are derived either analytically from the SDF gradient or via finite differences in volume data.

## Subtopics

### [Phong Shading with SDF Normals](1977-phong-shading-with-sdf-normals.md)
Calculates lighting by accumulating terms based on the Phong reflection model, using normals approximated from the SDF gradient. Choose this when specular highlights are required alongside diffuse shading.

### [Lambertian Lighting in SDF](2023-lambertian-lighting-in-sdf.md)
Computes diffuse illumination for SDF surfaces by evaluating the dot product between the surface normal and light direction. It solves the problem of shading ray-marched geometry without explicit mesh topology by deriving normals via finite differences. Unlike [Sphere Tracing Basics](52-sphere-tracing-basics.md), which returns a flat base color, this method integrates light direction and color.

### [Blinn-Phong Shading in Volume Rendering](2809-blinn-phong-shading-in-volume-rendering.md)
Applies Blinn-Phong lighting to volume data during ray marching. It differs from pure Phong shading by using the half-vector for specular components and integrates front-to-back compositing with lighting. Choose this for volumetric data where surface normals are approximated via central differences of intensity gradients.

### [Gouraud Shading](4645-gouraud-shading.md)
Computes colors for each vertex based on a lighting model and vertex normals, then linearly interpolates colors across the facet. Choose this for performance-critical applications where per-fragment normal calculation is too expensive, though it may exhibit shading artifacts on large polygons.

### Flat Shading
#### definition
A shading model assuming uniform lighting where light surrounds an object equally in all directions [S1, unofficial].
#### code
`glShadeModel(GL_FLAT)` [S1, unofficial]
#### pitfall
Flat shading diminishes the three dimensionality of the scene [S1, unofficial].

## Background
Diffuse lighting models originate from early computer graphics shading techniques, specifically within the OpenGL specification which defined flat and Gouraud (smooth) shading models [S1, unofficial]. The Lambertian model, named after Johann Heinrich Lambert, is the standard for ideal diffuse reflection. In the context of volume rendering and SDFs, these models were adapted to work with implicit surfaces where normals are derived from gradients rather than mesh topology.

## Sources
[S1] Smokeview (Version 6) A Tool for Visualizing Fire Dynamics Simulation Data Volume II : Technical Reference Guide, https://doi.org/10.6028/nist.sp.1017-2, fetched 2026-10-26 (unofficial)
