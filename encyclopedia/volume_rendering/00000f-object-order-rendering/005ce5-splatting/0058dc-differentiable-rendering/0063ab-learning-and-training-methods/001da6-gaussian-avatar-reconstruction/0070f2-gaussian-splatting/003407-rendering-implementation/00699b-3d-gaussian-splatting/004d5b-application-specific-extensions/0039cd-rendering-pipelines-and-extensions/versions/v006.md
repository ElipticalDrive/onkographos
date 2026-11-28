# Rendering Pipelines and Extensions

Also called: Gaussian Splatting Rendering Pipelines, Differentiable Gaussian Rasterization

## Overview
This family defines the computational mechanisms that convert a set of 3D Gaussian primitives (or their variants) into 2D images by projecting them onto the camera plane and compositing them [G]. It solves the problem of efficiently rasterizing soft, view-dependent primitives while supporting inverse rendering tasks such as lighting estimation and physical parameter optimization [G]. What sets these methods apart from standard rasterization is the use of alpha blending or weighted-sum compositing over sorted or spatially hashed primitives, often augmented with volume-based occlusion or explicit geometric intersection for sharp features [G].

## Shared principles
All methods in this family share the core mechanism of projecting 3D primitives into 2D screen space and accumulating color based on depth order or spatial locality [G].

1. **Projection:** Primitives are transformed from world space to camera space [G]. The 3D covariance matrix $\Sigma$ is transformed using the view matrix $W$ and the Jacobian $J$ of the perspective projection approximation: $\Sigma_c = J W \Sigma W^T J^T$ [G]. The 2D covariance $\Sigma_{2D}$ is extracted as the upper-left $2\times2$ block [G].
2. **Rasterization:** Primitives are assigned to screen tiles [G]. Visibility is determined by bounding boxes or projected extents [G].
3. **Compositing:** Color is accumulated using front-to-back alpha blending or order-independent techniques [G]. The final intensity for a pixel is $I_f = \sum_{i=1}^N c_i \tilde{\sigma}_i \tilde{\mathcal{G}}_i(\mathbf{p}) \prod_{j=1}^{i-1} (1 - \tilde{\sigma}_j \tilde{\mathcal{G}}_j(\mathbf{p}))$, where $c_i$ is color, $\tilde{\sigma}_i$ is opacity or density, and $\tilde{\mathcal{G}}_i(\mathbf{p})$ is the 2D Gaussian value at pixel $\mathbf{p}$ [G].
4. **Sorting:** Most methods require sorting primitives by depth within tiles to ensure correct front-to-back compositing, although some variants use spatial hashing or cell-level sorting to approximate this or enable order-independent transparency [G].

## Subtopics

### [Gaussian Splatting Volume Rendering](2063-gaussian-splatting-volume-rendering.md)
The standard differentiable rendering technique that synthesizes images by projecting 3D Gaussians and alpha-compositing them. It replaces triangle primitives with Gaussian splats to enable efficient tile-based rasterization and differentiable optimization. Choose this for high-fidelity novel view synthesis where view-dependent color (via spherical harmonics) is required.

### [Rendering and Lighting Pipelines](4135-rendering-and-lighting-pipelines.md)
Defines the general differentiable rendering pipeline for converting 3D Gaussian primitives into 2D images. It serves as the foundational architecture for most 3DGS implementations, handling projection, rasterization, and compositing. Choose this as the base implementation for standard 3DGS rendering.

### [Volume Encoding Gaussians](6093-volume-encoding-gaussians.md)
Adapts 3DGS for volume data visualization by removing per-Gaussian color and opacity attributes, training only scalar values. This allows the same underlying scalar field to be visualized with arbitrary transfer functions without retraining. Choose this for scientific visualization of volumetric data (e.g., medical scans, fluids).

### [BRDF and Relighting Extensions](6227-brdf-and-relighting-extensions.md)
Augments standard 3DGS by replacing baked view-dependent color with physically based rendering (PBR) parameters (albedo, roughness, normal). This enables dynamic relighting and material decomposition. Choose this when dynamic lighting or material analysis is required.

### [XPR Rendering Pipeline](17626-xpr-rendering-pipeline.md)
A framework that decouples method-specific logic from the shared rendering pipeline, allowing new methods to be implemented in a few hundred lines of code. It separates core mechanics (parallelization, tiling) from primitive evaluation, enabling portable execution across GPUs, TPUs, and CPUs via the XLA compiler. Choose this for cross-platform compatibility or rapid prototyping of new primitive types.

## Background
The rendering pipelines for 3D Gaussian Splatting originated from the need to render millions of Gaussian primitives in real-time, a task for which traditional triangle-based rasterizers are inefficient due to the soft, overlapping nature of the primitives. The core algorithm was introduced in the seminal 3D Gaussian Splatting paper [2812-3d-gaussian-splatting.md], which established the projection and alpha-compositing pipeline. Subsequent extensions have adapted this pipeline for specific domains: volume rendering [6093-volume-encoding-gaussians.md], physical relighting [6227-brdf-and-relighting-extensions.md], and cross-platform deployment [17626-xpr-rendering-pipeline.md]. The pipeline's efficiency relies on the affine approximation of the perspective projection and tile-based sorting, which allows for parallel execution on GPUs.

## Sources
[S1] Gaussian Splatting Volume Rendering, https://github.com/graphdeco-inria/gaussian-splatting, 2023-10-01
[S2] 3D Gaussian Splatting for Real-Time Radiance Field Rendering, Kerbl et al., ACM TOG, 2023
