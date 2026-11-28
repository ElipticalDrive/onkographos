# Lighting, Shadowing & Relighting

Also called: Deferred Surfel Rendering, Self-Shadowing, Light Space Rasterization

## Overview
This family extends 3D Gaussian Splatting rasterization to support dynamic lighting, physically-based relighting, and self-shadowing [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md, 4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md]. It solves the limitation of baked lighting by separating material properties from illumination, allowing real-time environment map swapping and physically plausible shadows within the Gaussian volume. Unlike forward-shading methods that bake lighting into spherical harmonics, these techniques use deferred shading or light-space transmittance accumulation to decouple geometry from lighting [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md].

## Shared principles
All methods in this family operate on 3D Gaussian primitives defined by position, covariance (or quaternion/scale), opacity, and color/SH coefficients. They extend the standard rasterization pipeline by introducing additional per-splat attributes or auxiliary rendering passes.

1.  **Material Extension:** For relighting, splats must carry physical material attributes: normal, roughness, reflectance (specular strength), and albedo (diffuse color) [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md]. These are stored in a parallel Material Buffer aligned with the Splat Buffer [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md].
2.  **Light-Space Projection:** For self-shadowing, a virtual "sun camera" is constructed to simulate a directional light source [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md]. This camera is placed at a large distance with a narrow field of view (FOV) to minimize parallax error to below 2% [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].
3.  **Depth Sorting:** Both relighting (for cache locality) and shadowing (for correct front-to-back accumulation) require sorting. Relighting uses spatial locality algorithms like Morton codes [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md]. Shadowing sorts Gaussians by depth relative to the sun camera [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].
4.  **Covariance Handling:** During the light pass for shadowing, the projection covariance is not expanded by $0.3I$ and no amplitude compensation is applied, preserving the main camera pass's sampling expansion [4487-3d-gaussian-splatting-self-shadowing-t_light.md].
5.  **Gradient Treatment:** In shadowing calculations, sun camera framing, sorting, coverage, and threshold branches are treated as constants to prevent incorrect gradient propagation [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].

## Subtopics

### [Ref-Gaussian Deferred Surfel Rendering with Split-Sum IBL](4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md)
A real-time deferred shading method for relighting 3DGS models on mobile hardware (iOS/Metal) [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md]. It differs from forward-shading by decomposing splats into PBR materials and using a deferred G-Buffer with split-sum environment map integration [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md]. Choose this for dynamic environment lighting without retraining. It adapts splat-based rasterization to the deferred workflow [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md].

### [3D Gaussian Splatting Self-Shadowing (T_light)](4487-3d-gaussian-splatting-self-shadowing-t-light.md)
Calculates self-shadowing transmittance $T_{light}$ by tracking front occlusion from the light source's perspective [4487-3d-gaussian-splatting-self-shadowing-t_light.md]. It differs from standard alpha blending by explicitly modeling volume occlusion [4487-3d-gaussian-splatting-self-shadowing-t_light.md]. Choose this for physically plausible shadows within the Gaussian volume [4487-3d-gaussian-splatting-self-shadowing-t_light.md].

### [Light Space Rasterization for T_light](4489-light-space-rasterization-for-t-light.md)
Computes $T_{light}$ via light-space rasterization, providing full continuous gradients and zero parameter expansion [4489-light-space-rasterization-for-t_light.md]. It differs from voxel-based fallbacks by enabling differentiable relighting without specific training hyperparameters [4489-light-space-rasterization-for-t_light.md]. Choose this for differentiable shadow integration [4489-light-space-rasterization-for-t_light.md].

## Background
The base 3D Gaussian Splatting method [2812-3d-gaussian-splatting-rasterization.md] primarily supports baked or view-dependent lighting via spherical harmonics. These extensions were developed to enable dynamic relighting and self-shadowing, addressing the need for physically based rendering in real-time applications [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md, 4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md]. Ref-Gaussian targets mobile hardware efficiency [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md], while the $T_{light}$ methods focus on differentiable shadow computation [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].

## Sources
