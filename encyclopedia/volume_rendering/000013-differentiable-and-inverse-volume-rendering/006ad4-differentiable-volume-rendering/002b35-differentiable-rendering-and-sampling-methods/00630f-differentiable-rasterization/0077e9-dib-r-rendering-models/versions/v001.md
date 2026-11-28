# DIB-R Rendering Models
Also called: DIB-R

## Overview
DIB-R rendering models define a framework where pixel values are determined by interpolating abstract vertex attributes across rasterized triangles, followed by the application of local illumination models in the fragment shader [S1, unofficial]. What sets this approach apart from its siblings is its explicit support for back-propagating gradients through the interpolation of diverse vertex attributes (normals, light/eye directions, material properties) to enable fully differentiable rendering of meshes with complex lighting, rather than relying solely on color or opacity interpolation [S1, unofficial].

## Shared principles
The core mechanism relies on defining pixel values $I_i$ as a function of interpolated abstract vertex attributes $u_0, u_1, u_2$ [S1, unofficial]. The renderer expects a mesh input, where vertex position is a primary attribute, but simultaneously supports a large array of other vertex attributes that are interpolated during rasterization and back-propagated through [S1, unofficial].

The rendering process is split into two stages:
1.  **Interpolation:** Light-related attributes (normals, light directions, eye directions) and material properties are interpolated across the triangle surface during rasterization [S1, unofficial].
2.  **Fragment Shader Application:** Specific lighting models are applied in the fragment shader using these interpolated attributes to compute the final color [S1, unofficial].

The vertex attributes supported for interpolation include:
*   Vertex position
*   Vertex color
*   Texture coordinates ($u, v$)
*   Normals
*   Light directions
*   Eye directions
*   Material properties (diffuse $k_d$, specular $k_s$, shininess $\alpha$) [S1, unofficial]

## Subtopics

### Basic Models
Renders the image directly using vertex colors or texture maps defined by $u, v$ coordinates over a learned or predefined texture. This model does not apply complex lighting effects, relying instead on the interpolated color or texture value. See [Basic Models](18491-dib-r-basic-models.md).

### Phong and Lambertian Models
Applies local illumination models. The Phong model computes image color $I$ using vertex normals, light directions, eye directions, and material properties. The Lambertian model is a simplification focusing on diffuse reflection. These models merge the mesh color with lighting factors derived from the interpolated attributes. See [Phong and Lambertian Models](18492-dib-r-phong-lambertian.md).

## Background
The DIB-R (Differentiable Interpolation-based Renderer) framework was introduced by Liu et al. in the paper "Learning to Predict 3D Objects with an Interpolation-based Differentiable Renderer" [S1, unofficial]. The work aims to create a fully differentiable rendering pipeline where gradients can be analytically determined for all aspects of the rasterization process, including lighting and material properties, by back-propagating through the interpolated vertex attributes [S1, unofficial].

## Sources
[S1] Learning to Predict 3D Objects with an Interpolation-based Differentiable Renderer, https://doi.org/10.48550/arxiv.1908.01210, fetched 2026-10-26 (unofficial)
