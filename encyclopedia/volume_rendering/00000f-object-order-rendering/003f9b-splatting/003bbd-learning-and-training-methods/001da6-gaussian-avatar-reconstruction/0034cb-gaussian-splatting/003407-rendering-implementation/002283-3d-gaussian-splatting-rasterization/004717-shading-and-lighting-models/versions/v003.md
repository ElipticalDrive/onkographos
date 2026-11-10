# Shading and Lighting Models

Also called: Gaussian Splatting Shading, 3DGS Lighting, Deferred Gaussian Shading

## Overview
Shading and Lighting Models extend the standard 3D Gaussian Splatting rasterization pipeline to support physically based rendering (PBR), self-occlusion, and anti-aliased forward shading. While standard splatting assumes a fixed, uniform lighting environment and simple alpha blending, this family introduces mechanisms for deferred shading pipelines, light-space rasterization for shadows, and analytic integration approximations for multiscale fidelity. It differs from sibling methods like [Analytic-Splatting Forward Shading](3459-analytic-splatting-forward-shading.md) by focusing on material attributes and light interaction rather than just pixel-area integration, and from [Transparency and Blending Models](9274-transparency-and-blending-models.md) by explicitly handling light source geometry and PBR material buffers.

## Shared principles
All methods in this family operate on 3D Gaussian primitives defined by mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$ (or scales and orientation), opacity $o$, and color $\mathbf{c}$ (or spherical harmonic coefficients). The core challenge is computing the pixel color $C$ and accumulated alpha $\alpha$ given a camera ray. Standard splatting projects 3D Gaussians to 2D ellipses, computes a 2D alpha mask, and blends them in screen space. These models modify the blending or shading stage: some refine the depth metric used for sorting, others integrate the 3D density function directly along the ray, and others use weighted averages to bypass sorting. All assume the Gaussians are sorted or processed in a manner that approximates front-to-back accumulation to ensure correct occlusion.

For relighting, splats must carry physical material attributes: normal, roughness, reflectance (specular strength), and albedo (diffuse color) [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md]. These are stored in a parallel Material Buffer aligned with the Splat Buffer [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md].

For self-shadowing, a virtual "sun camera" is constructed to simulate a directional light source [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md]. This camera is placed at a large distance with a narrow field of view (FOV) to minimize parallax error to below 2% [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].

Both relighting (for cache locality) and shadowing (for correct front-to-back accumulation) require sorting. Relighting uses spatial locality algorithms like Morton codes [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md]. Shadowing sorts Gaussians by depth relative to the sun camera [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].

During the light pass for shadowing, the projection covariance is not expanded by $0.3I$ and no amplitude compensation is applied, preserving the main camera pass's sampling expansion [4487-3d-gaussian-splatting-self-shadowing-t_light.md]. In shadowing calculations, sun camera framing, sorting, coverage, and threshold branches are treated as constants to prevent incorrect gradient propagation [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].

## Subtopics

### [Analytic-Splatting Forward Shading](3459-analytic-splatting-forward-shading.md)
Computes the 2D Gaussian signal response for a pixel by performing eigendecomposition on the projected covariance matrix, rotating the coordinate system to decouple axes, and applying a conditioned logistic function to approximate the pixel area integral of the Gaussian CDF. It differs from standard 3D Gaussian Splatting by replacing the simple Gaussian evaluation at the pixel center with an analytic integration approximation, and from Mip-Splatting by using a logistic function approximation rather than a Mip-NeRF-inspired cone integral, thereby achieving state-of-the-art detail fidelity under multiscale and super-resolution settings at the cost of increased computational operations. Choose this when high-frequency detail preservation is critical and computational budget allows for eigendecomposition per pixel.

### [Transparency and Blending Models](9274-transparency-and-blending-models.md)
Define how 3D Gaussian primitives are composited into a 2D image, addressing the limitations of standard front-to-back alpha blending which requires strict depth sorting and assumes non-overlapping primitives. These models range from hybrid approaches that balance performance and accuracy to analytic volumetric integration that eliminates projection artifacts, and order-independent methods that bypass sorting entirely. Choose this when correct occlusion handling is required without the overhead of strict sorting or when dealing with highly overlapping primitives.

### [Lighting, Shadowing & Relighting](11894-lighting-shadowing-relighting.md)
Extends 3D Gaussian Splatting to support physically based lighting, self-occlusion, and environment reflections. It solves the limitation of standard splatting, which assumes a fixed, uniform lighting environment, by introducing deferred shading pipelines for relighting and dedicated light-space rasterization passes for self-shadowing. Unlike sibling methods that may use simplified shading functions or voxel-based fallbacks, these approaches provide full PBR material attributes and continuous differentiable gradients for shadows without modifying the primary camera's covariance expansion. Choose this when dynamic lighting, shadows, or environment reflections are required.

## Background
The standard 3D Gaussian Splatting method [S1] assumes a fixed lighting environment. Extensions to support PBR and shadows emerged to enable interactive relighting and realistic scene synthesis. Deferred shading pipelines for Gaussian Splatting were introduced to handle complex lighting [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md], while light-space rasterization techniques were developed to compute self-occlusion [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md]. Analytic approximations for splatting were developed to improve anti-aliasing and super-resolution performance [S1].

## Sources
[S1] Analytic-Splatting Forward Shading, https://github.com/ashawkey/analytic-splatting, 2024
