# Rendering Pipelines and Extensions

Also called: 3DGS Rendering Extensions, Gaussian Splatting Pipeline Variants

## Overview
This family encompasses the differentiable rendering mechanisms and architectural extensions that convert 3D Gaussian primitives into 2D images, enabling both standard novel view synthesis and advanced inverse rendering tasks [G]. It solves the problem of efficiently rasterizing soft, view-dependent volumetric primitives while supporting complex appearance models such as physical lighting and volume data visualization [G]. What sets these methods apart from standard [3D Gaussian Splatting](1795-3d-gaussian-splatting.md) is the decoupling of geometric representation from appearance, allowing for explicit material properties (BRDF) or scalar density fields (VEG) rather than baked spherical harmonics [S1, S2, S1, unofficial].

## Shared principles
All methods in this family share the core mechanism of projecting 3D primitives into 2D screen space and accumulating color based on depth order or spatial locality [G].

1.  **Projection:** Primitives are transformed from world space to camera space [G].
2.  **Rasterization:** Primitives are assigned to screen tiles [G]. Visibility is determined by bounding boxes or projected extents [G].
3.  **Compositing:** Color is accumulated using front-to-back alpha blending or order-independent techniques [G].
4.  **Sorting:** Most methods require sorting primitives by depth within tiles to ensure correct front-to-back compositing, although some variants use spatial hashing or cell-level sorting to approximate this or enable order-independent transparency [G].

## Subtopics

### [Gaussian Splatting Volume Rendering](2063-gaussian-splatting-volume-rendering.md)
The standard differentiable rendering technique that synthesizes images by projecting 3D Gaussians and alpha-compositing them. It solves the problem of high-fidelity, real-time novel view synthesis by replacing triangle meshes with volumetric ellipsoids. Unlike NeRFs, it uses explicit primitives for efficient tile-based rasterization.

### [Rendering and Lighting Pipelines](4135-rendering-and-lighting-pipelines.md)
Defines the differentiable pipelines that convert 3D Gaussians into 2D images. It solves the problem of efficiently rasterizing soft primitives while supporting inverse rendering tasks. It differs from standard rasterization by using alpha blending over sorted primitives, often augmented with volume-based occlusion [G].

### [Volume Encoding Gaussians](6093-volume-encoding-gaussians.md)
Adapts 3DGS for volume data visualization by decoupling geometry from appearance. Unlike standard 3DGS which optimizes color and opacity, VEG trains only scalar density values, enabling interactive transfer function application post-training.

### [BRDF and Relighting Extensions](6227-brdf-and-relighting-extensions.md)
Extends 3DGS by decoupling material properties from lighting for physically based rendering. Unlike standard 3DGS which bakes view-dependent appearance into spherical harmonics, these extensions store explicit BRDF parameters (albedo, roughness, normals) per Gaussian, allowing dynamic relighting.

## Background
The rendering pipelines for 3D Gaussian Splatting originated from the foundational work on 3DGS, which introduced the tile-based rasterization of 3D Gaussians for real-time novel view synthesis [S1]. The extension to volume encoding and BRDF-based relighting represents a shift from implicit neural fields and baked appearance to explicit, physically interpretable primitives [S1, S2, unofficial]. These methods are primarily used in computer graphics for real-time rendering, digital twins, and scientific visualization [S1][S2, unofficial].

## Sources
[S1] Kerbl, B., Kopanas, G., Leimkühler, T., & Drettakis, G. (2023). 3D Gaussian Splatting for Real-Time Radiance Field Rendering. ACM Transactions on Graphics (TOG).
[S2] 3D Gaussian Splatting Implementation Notes. (unofficial)
[S3] View-Dependent Color Analysis in Gaussian Splatting. (unofficial)
