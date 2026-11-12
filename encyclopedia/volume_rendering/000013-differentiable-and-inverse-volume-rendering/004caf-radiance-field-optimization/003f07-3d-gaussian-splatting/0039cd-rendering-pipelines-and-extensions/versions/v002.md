# Rendering Pipelines and Extensions

Also called: 3DGS Rendering Extensions, Gaussian Splatting Pipeline Variants

## Overview
This family encompasses the differentiable rendering architectures and material extensions that convert 3D Gaussian primitives into 2D images, enabling both standard novel view synthesis and advanced inverse rendering tasks [G]. It solves the problem of efficiently rasterizing soft, view-dependent primitives while supporting complex appearance models such as physically based rendering (PBR) and volume visualization [G]. What sets these methods apart from standard triangle-based rasterization is the use of analytic Gaussian projection and alpha compositing, often augmented with specific data structures for density control or material decoupling [G].

## Shared principles
All methods in this family share the core mechanism of projecting 3D primitives into 2D screen space and accumulating color based on depth order or spatial locality [G].

1.  **Projection:** Primitives are transformed from world space to camera space [G].
2.  **Rasterization:** Primitives are assigned to screen tiles [G]. Visibility is determined by bounding boxes or projected extents [G].
3.  **Compositing:** Color is accumulated using front-to-back alpha blending or order-independent techniques [G].
4.  **Sorting:** Most methods require sorting primitives by depth within tiles to ensure correct front-to-back compositing, although some variants use spatial hashing or cell-level sorting to approximate this or enable order-independent transparency [G].

## Subtopics

### [Gaussian Splatting Volume Rendering](2063-gaussian-splatting-volume-rendering.md)
A technique that synthesizes images by projecting 3D Gaussian distributions onto the image plane and alpha-compositing them along viewing rays. It solves the problem of high-quality, real-time novel view synthesis by replacing traditional triangle-based rasterization with smooth, differentiable Gaussian primitives. Unlike sibling methods that may use neural fields or explicit meshes, this method relies on the analytic projection of 2D Gaussians and a tile-based rasterization strategy for parallelization.

### [Rendering and Lighting Pipelines](4135-rendering-and-lighting-pipelines.md)
This family defines the differentiable rendering pipelines that convert a set of 3D Gaussian primitives (or their variants) into 2D images by projecting them onto the camera plane and compositing them [G]. It solves the problem of efficiently rasterizing soft, view-dependent primitives while supporting inverse rendering tasks such as lighting estimation and physical parameter optimization [G]. What sets these methods apart from standard rasterization is the use of alpha blending or weighted-sum compositing over sorted or spatially hashed primitives, often augmented with volume-based occlusion or explicit geometric intersection for sharp features [G].

### [Volume Encoding Gaussians](6093-volume-encoding-gaussians.md)
Volume Encoding Gaussians (VEG) adapts the 3D Gaussian Splatting framework for volume data visualization by decoupling the geometric representation from the appearance model. Unlike standard 3D Gaussian Splatting which optimizes color and opacity per primitive, VEG trains only scalar density values, enabling interactive transfer function application post-training. This distinguishes it from [Gaussian Splatting Volume Rendering](2063-gaussian-splatting-volume-rendering.md), which typically retains explicit opacity and color attributes for direct rendering.

### [BRDF and Relighting Extensions](6227-brdf-and-relighting-extensions.md)
These methods extend standard 3D Gaussian Splatting (3DGS) by decoupling material properties from lighting, enabling physically based rendering (PBR) and dynamic relighting. Unlike standard 3DGS, which bakes view-dependent appearance into spherical harmonics (SH) coefficients under an emission-absorption model, these extensions store explicit BRDF parameters (albedo, roughness, normals) per Gaussian, allowing the scene to be illuminated by arbitrary light sources and environment maps.

## Background
The rendering pipelines for 3D Gaussian Splatting originated with the foundational 3DGS paper, which introduced the tile-based rasterization and differentiable projection framework [S1]. The extension to volume encoding and BRDF-based relighting represents a shift from appearance-based optimization to physically-based and data-driven representations, allowing for greater flexibility in visualization and inverse rendering tasks [S1, S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting for Real-Time Radiance Field Rendering, Kerbl et al., 2023
[S2] 3D Gaussian Splatting Implementation Details, Unofficial Documentation, 2023
