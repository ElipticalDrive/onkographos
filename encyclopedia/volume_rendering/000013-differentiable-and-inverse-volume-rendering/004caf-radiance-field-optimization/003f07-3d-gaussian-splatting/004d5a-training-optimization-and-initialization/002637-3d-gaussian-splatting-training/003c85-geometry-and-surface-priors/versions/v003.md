# Geometry and Surface Priors

Also called: Geometric regularization, Surface alignment, Depth priors

## Overview
Geometry and Surface Priors address the inherent ambiguity and lack of explicit surface structure in standard 3D Gaussian Splatting (3DGS), which produces floating ellipsoids as an emergent property rather than a defined mesh [S1]. This family of methods enforces geometric consistency by coupling the discrete Gaussian representation with continuous geometric signals, such as Signed Distance Functions (SDFs) or monocular depth estimates [S1]. Unlike [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md), which relies solely on photometric rendering loss, these techniques explicitly constrain Gaussian positions, orientations, and scales to align with known or estimated surface geometry [S1].

## Shared principles
All methods in this family share the goal of transferring explicit surface properties to the discrete Gaussian representation [S1]. The core mechanism involves computing a consistency loss between the Gaussian attributes and a geometric prior.

1.  **Geometric Prior Source:** The prior is either an implicit function (e.g., a neural SDF network $f_\theta(\mathbf{x})$) or an explicit map (e.g., monocular depth $D^{est}$) [S1].
2.  **SDF Consistency:** To enforce that Gaussians lie on the surface, the loss encourages the signed distance value at each Gaussian center $\mathbf{c}_i$ to be zero:
    $$ \mathcal{L}_{sdf} = \frac{1}{N} \sum_{i=1}^{N} |f_\theta(\mathbf{c}_i)| $$
    [S1]
3.  **Normal Consistency:** To align the Gaussian orientation with the surface geometry, the loss minimizes the angular difference between the Gaussian normal $\mathbf{n}_i$ (derived from the covariance matrix or rotation quaternion) and the geometric normal $\mathbf{m}_i$ (derived from the SDF gradient or depth map):
    $$ \mathcal{L}_{normal} = \frac{1}{N} \sum_{i=1}^{N} (1 - \mathbf{n}_i \cdot \mathbf{m}_i) $$
    [S1]
4.  **Optimization:** These losses are added to the standard photometric rendering loss $\mathcal{L}_{rgb}$ to form the total objective:
    $$ \mathcal{L}_{total} = \mathcal{L}_{rgb} + \lambda_{sdf} \mathcal{L}_{sdf} + \lambda_{normal} \mathcal{L}_{normal} + \dots $$
    [S1]

## Subtopics

### [NeuSG](9172-neusg.md)
[NeuSG](9172-neusg.md) integrates the NeuS neural surface reconstruction method into the 3DGS framework. It is chosen when high-fidelity mesh extraction is required directly from the Gaussian optimization process. Unlike vanilla 3DGS, it couples a continuous SDF with the Gaussian rasterizer.

### [Monocular Depth Scale Alignment](9183-monocular-depth-scale-alignment.md)
[Monocular Depth Scale Alignment](9183-monocular-depth-scale-alignment.md) resolves the up-to-scale ambiguity in image-only reconstruction pipelines. It aligns the metric scale of optimized Gaussians with metric-scale monocular depth estimates by optimizing a single global scale factor $a$. Unlike sibling methods that prune points, this method fixes Gaussian parameters and optimizes only the scale.

### [Sparse-view 3D Gaussian Splatting with Depth Priors](9185-sparse-view-3d-gaussian-splatting-with-depth-priors.md)
This method addresses sparse-view reconstruction by integrating pre-trained 2D depth estimation networks as geometric priors. It distinguishes itself from [NeuSG](9172-neusg.md) and [GSDF](9205-gsdf.md) by relying on explicit depth maps from a fixed 2D network rather than learning an implicit SDF. It enhances fidelity by removing Gaussians with inconsistent depth values and utilizing Score Distillation Sampling (SDS).

### [GSDF](9205-gsdf.md)
[GSDF](9205-gsdf.md) is a geometry regularization technique that enhances surface reconstruction by enforcing consistency between the discrete Gaussian representation and a continuous SDF. It differs from siblings like NeuSG by specifically leveraging neural SDFs to recover detailed mesh structures, adapting strategies from NeRF-based methods.

### [Surface and Geometry Alignment](9267-surface-and-geometry-alignment.md)
[Surface and Geometry Alignment](9267-surface-and-geometry-alignment.md)

#### GaussianPro
- [S1] A method that considers neighboring views' normal direction consistency to produce better reconstruction results.
- [S1] Introduces a progressive propagation strategy for updating Gaussians, considering normal consistency between neighboring views.
- [S1] Adds plane constraints to guide the growth of 3D Gaussian Splatting.

#### 3DGSR
- [S1] A method that encourages consistency between Signed Distance Function (SDF) and 3DGS to enhance geometry reconstruction quality.

## Background
The concept of transferring surface properties to 3DGS originated from the need to mitigate the "popping" artifacts and lack of explicit geometry in standard 3DGS [S1]. NeuSG was one of the first to couple a neural SDF with 3DGS [S1]. Subsequently, methods like GSDF and 3DGSR expanded on this by encouraging SDF consistency [S1]. More recently, approaches like Monocular Depth Scale Alignment and sparse-view methods have incorporated external priors from monocular depth estimators to resolve scale ambiguities and improve reconstruction from limited views [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
