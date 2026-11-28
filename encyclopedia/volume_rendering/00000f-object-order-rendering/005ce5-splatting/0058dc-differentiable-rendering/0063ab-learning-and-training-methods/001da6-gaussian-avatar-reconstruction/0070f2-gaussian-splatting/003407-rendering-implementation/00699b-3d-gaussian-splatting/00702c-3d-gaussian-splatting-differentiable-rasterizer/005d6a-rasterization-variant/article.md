# Rasterization Variant


## Overview
This article defines the family of differentiable rasterization methods that project 3D Gaussians to 2D screen space and accumulate color per pixel using alpha compositing, allowing gradients to flow back to Gaussian parameters (position, scale, rotation, opacity, color). It builds on [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md). The family is distinguished by variations in how the projection, culling, blending, and auxiliary outputs are handled to support specific rendering pipelines (e.g., X-ray, hybrid 3D/2D, or mixed geometry).

## Shared principles
All variants in this family share the core mechanism of differentiable rasterization for 3D Gaussians. The common mathematical foundation involves:
1.  **Projection:** Transforming 3D Gaussian parameters (mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$) into 2D screen-space ellipses using the Jacobian of the projection transformation.
2.  **Sorting:** Sorting Gaussians by depth within tiles to ensure correct front-to-back rendering order.
3.  **Alpha Compositing:** Accumulating color $C$ and transmittance $T$ for each pixel using the standard alpha blending equations:
    $$ T_{i+1} = T_i (1 - \alpha_i) $$
    $$ C_{i+1} = C_i + \alpha_i T_i \mathbf{c}_i $$
    where $\alpha_i$ is the 2D alpha value derived from the projected Gaussian's covariance and opacity, and $\mathbf{c}_i$ is the color.
4.  **Differentiability:** The entire pipeline is differentiable with respect to Gaussian attributes, enabling gradient-based optimization.

## Subtopics

### [Gaussian Splat Rasterization (Vertex Shader)](3253-gaussian-splat-rasterization-vertex-shader.md)
Implements the vertex shader stage of a GPU-accelerated rasterizer. It computes 2D screen-space projections and basis vectors for instanced quad rendering. It differs from CUDA-based variants by executing calculations in the vertex shader, relying on subsequent geometry/fragment shaders for accumulation. Choose this when using a traditional graphics pipeline with vertex/geometry shaders rather than a custom CUDA kernel.

### [Masked-Rasterization](4366-masked-rasterization.md)
Allows "masked" (probabilistically inactive) 3D Gaussians to participate in the rasterization pipeline and receive gradient updates. It solves the problem of outdated Gaussians in dynamic pruning by integrating binary existence masks into the transmittance and color accumulation steps. Choose this when performing dynamic Gaussian pruning or training with probabilistic existence masks.

### [Differentiable Radiative Rasterization](4526-differentiable-radiative-rasterization.md)
A GPU-accelerated method for rendering novel X-ray projections from radiative 3D Gaussian point clouds. It models radiation intensity rather than visible light color, computing scalar intensity values based on the probability density of Gaussians along the X-ray path. Choose this for X-ray imaging or volume rendering applications requiring differentiable X-ray synthesis.

### [3D/2D Hybrid Gaussian Rasterization with Fallback Sigma](4812-3d-2d-hybrid-gaussian-rasterization-with-fallback-sigma.md)
Computes alpha attenuation using a hybrid sigma metric that combines a 3D ray-surfel distance estimate with a 2D screen-space fallback bound. It solves inaccurate alpha estimation in standard rasterization when the 2D covariance projection is ill-conditioned or the Gaussian is far from the camera. Choose this when robust alpha estimation is required for distant or poorly conditioned Gaussians.

### [Gaussian Rasterization with F-Count](17250-gaussian-rasterization-with-f-count.md)
Computes an auxiliary output: the number of times each 3D Gaussian is activated (rendered) during the forward pass, alongside standard color and radii. It sets itself apart by providing explicit per-Gaussian activation counts (`gaussians_count`) and an `important_score`. Choose this for Gaussian pruning or importance sampling tasks that require tracking Gaussian visibility.

### [Single-Pass Rasterization with Alpha-Blending](19202-single-pass-rasterization-with-alpha-blending.md)
Provides a unified rendering pipeline that rasterizes both 3D Gaussians and triangle meshes in a single pass using alpha-blending. It ensures correct spatial occlusion and color blending without dropping rendering rates. Choose this when rendering mixed scenes containing both Gaussian splats and triangle meshes.

#### Analytic Tau Rasterization
A rasterization branch (`use_analytic_tau`) that computes the per-pixel accumulated analytic line-integral optical thickness $\tau$ along the view ray. Alpha is derived physically as $\text{alpha} = 1 - \text{exp}(-\tau)$ using Beer-Lambert extinction, rather than using heuristic alpha values [S1, unofficial].

## Background
The family originates from the 3D Gaussian Splatting (3DGS) framework, which introduced differentiable rasterization for 3D Gaussians [G]. The specific variants listed here have been developed to address limitations in the original 3DGS rasterizer, such as handling dynamic pruning (Masked-Rasterization), supporting different modalities like X-rays (DRR), improving robustness (Hybrid Sigma), and enabling mixed geometry rendering (Single-Pass). The "Analytic Tau" variant is associated with the 4DGS-Volume-Cloud project, which extends 3DGS to volumetric rendering with analytic optical thickness [S1, unofficial].

## Sources
[S1] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
