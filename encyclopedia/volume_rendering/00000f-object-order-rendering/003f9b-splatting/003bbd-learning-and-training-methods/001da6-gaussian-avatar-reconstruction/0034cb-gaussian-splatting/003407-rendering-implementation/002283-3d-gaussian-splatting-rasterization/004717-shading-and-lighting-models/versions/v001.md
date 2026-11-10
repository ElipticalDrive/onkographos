# Shading and Lighting Models

Also called: Gaussian Shading, Gaussian Lighting, Gaussian Relighting

## Overview
Shading and Lighting Models define how the radiance of 3D Gaussian primitives is computed and composited into a final image, extending the basic rasterization of position and opacity to include color, transparency, and illumination. This family solves the limitations of baked lighting and simple alpha blending by providing mechanisms for anti-aliased forward shading, physically accurate transparency, and dynamic relighting with shadows. It sets itself apart from sibling rasterization articles by focusing on the per-pixel color computation and light interaction rather than the geometric projection or sorting algorithms.

## Shared principles
All methods in this family operate on 3D Gaussian primitives defined by mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$ (or scales and orientation), opacity $o$, and color $\mathbf{c}$ (or spherical harmonic coefficients). The core challenge is computing the pixel color $C$ and accumulated alpha $\alpha$ given a camera ray. Standard splatting projects 3D Gaussians to 2D ellipses, computes a 2D alpha mask, and blends them in screen space. These models modify the blending or shading stage: some refine the depth metric used for sorting, others integrate the 3D density function directly along the ray, and others use weighted averages to bypass sorting. All assume the Gaussians are sorted or processed in a manner that approximates front-to-back accumulation to ensure correct occlusion.

For relighting, splats must carry physical material attributes: normal, roughness, reflectance (specular strength), and albedo (diffuse color) [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md]. These are stored in a parallel Material Buffer aligned with the Splat Buffer [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md].

For self-shadowing, a virtual "sun camera" is constructed to simulate a directional light source [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md]. This camera is placed at a large distance with a narrow field of view (FOV) to minimize parallax error to below 2% [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].

Both relighting (for cache locality) and shadowing (for correct front-to-back accumulation) require sorting. Relighting uses spatial locality algorithms like Morton codes [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md]. Shadowing sorts Gaussians by depth relative to the sun camera [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].

During the light pass for shadowing, the projection covariance is not expanded by $0.3I$ and no amplitude compensation is applied, preserving the main camera pass's sampling expansion [4487-3d-gaussian-splatting-self-shadowing-t_light.md]. In shadowing calculations, sun camera framing, sorting, coverage, and threshold branches are treated as constants to prevent incorrect gradient propagation [4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].

## Subtopics

### [Analytic-Splatting Forward Shading](3459-analytic-splatting-forward-shading.md)
Computes color by analytically integrating the Gaussian PDF over the pixel area using a conditioned logistic function, solving aliasing artifacts inherent in point-sampling. Choose when anti-aliasing is critical and computational overhead is acceptable.

### [Transparency and Blending Models](9274-transparency-and-blending-models.md)
Defines compositing strategies for order-dependent alpha blending, addressing popping artifacts and transmittance inaccuracies. Choose when physical accuracy of transparency or order-independent transparency (OIT) is required.

### [Lighting, Shadowing & Relighting](11894-lighting-shadowing-relighting.md)
Extends rasterization to support dynamic lighting, physically-based relighting, and self-shadowing by separating material properties from illumination. Choose when real-time environment map swapping or shadow casting is needed.

## Background
The concept of shading 3D Gaussians emerged with the original 3D Gaussian Splatting paper [S1], which used spherical harmonics for view-dependent color. Subsequent work introduced analytic integration for anti-aliasing [S1], refined transparency models to handle sorting artifacts [2812-3d-gaussian-splatting-rasterization.md, 3294-hybrid-transparency.md, 4440-analytic-transmittance-3dgs.md, 7562-sparseoit-rasterizer.md], and extended the pipeline to deferred shading and light-space rasterization for shadows and relighting [4464-ref-gaussian-deferred-surfel-rendering-with-split-sum-ibl.md, 4487-3d-gaussian-splatting-self-shadowing-t_light.md, 4489-light-space-rasterization-for-t_light.md].

## Sources
[S1] Analytic-Splatting Forward Shading notes
