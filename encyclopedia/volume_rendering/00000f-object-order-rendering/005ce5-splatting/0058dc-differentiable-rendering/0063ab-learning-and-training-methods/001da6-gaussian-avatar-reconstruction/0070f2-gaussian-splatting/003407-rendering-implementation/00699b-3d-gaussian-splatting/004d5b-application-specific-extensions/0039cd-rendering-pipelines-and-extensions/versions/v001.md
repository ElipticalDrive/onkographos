# Rendering Pipelines and Extensions

Also called: Gaussian Splatting Rendering, 3DGS Pipelines, BRDF Extensions for 3DGS

## Overview
This family encompasses the differentiable rendering mechanisms and optimization extensions that convert 3D Gaussian primitives into 2D images. It solves the problem of efficiently rasterizing soft, view-dependent ellipsoids while supporting inverse rendering tasks such as lighting estimation and physical parameter optimization [G]. What sets these methods apart from standard rasterization is the use of alpha blending or weighted-sum compositing over sorted or spatially hashed primitives, often augmented with volume-based occlusion or explicit geometric intersection for sharp features [G].

## Shared principles
All methods in this family share the core mechanism of projecting 3D primitives into 2D screen space and accumulating color based on depth order or spatial locality [G].

1.  **Projection:** Primitives are transformed from world space to camera space [G].
2.  **Rasterization:** Primitives are assigned to screen tiles [G]. Visibility is determined by bounding boxes or projected extents [G].
3.  **Compositing:** Color is accumulated using front-to-back alpha blending or order-independent techniques [G].
4.  **Sorting:** Most methods require sorting primitives by depth within tiles to ensure correct front-to-back compositing, although some variants use spatial hashing or cell-level sorting to approximate this or enable order-independent transparency [G].

## Subtopics

### [Gaussian Splatting Volume Rendering](2063-gaussian-splatting-volume-rendering.md)
The standard differentiable rendering technique that synthesizes images by projecting 3D Gaussian distributions onto the image plane and alpha-compositing them. It reconstructs a 3D scene from pictures using Gaussian splats as primitives, materializing smooth ellipses in 2D from 3D probability distributions to capture view-dependent effects via spherical harmonics.

### [Rendering and Lighting Pipelines](4135-rendering-and-lighting-pipelines.md)
Defines the pipelines that convert 3D Gaussian primitives into 2D images by projecting them onto the camera plane and compositing them. It solves the problem of efficiently rasterizing soft, view-dependent primitives while supporting inverse rendering tasks [G].

### [Volume Encoding Gaussians](6093-volume-encoding-gaussians.md)
Adapts 3D Gaussian Splatting for volume data visualization by removing color and opacity attributes for each Gaussian, training only scalar values and enabling interactive transfer function application post-training.

### [BRDF and Relighting Extensions](6227-brdf-and-relighting-extensions.md)
Extends 3D Gaussian Splatting to support physically based rendering (PBR) and dynamic relighting by decoupling material properties from lighting conditions. Unlike standard 3DGS, which bakes lighting into spherical harmonics (SH) coefficients and limits performance on reflective surfaces, these methods store explicit BRDF parameters (albedo, roughness, normals) per Gaussian. This enables per-point relighting and specular reflection, addressing the inability of standard emission-absorption models to handle complex interreflections or material variation.

## Background
The core rendering pipeline for 3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1]. It established the tile-based rasterization approach for soft primitives. Subsequent work has extended this pipeline to handle volume data [S2, unofficial] and physically based materials [S1].

## Sources
[S1] Kerbl, B., Kopanas, G., Leimkühler, T., & Drettakis, G. (2023). 3D Gaussian Splatting for Real-Time Radiance Field Rendering. ACM Transactions on Graphics (TOG).
[S2] 3D Gaussian Splatting Technical Documentation and Implementation Notes. (unofficial)
