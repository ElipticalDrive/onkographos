# Geometry and Surface Priors

Also called: Surface Priors, Geometric Constraints, SDF-Gaussian Consistency

## Overview
Geometry and Surface Priors address the inherent lack of explicit surface structure in vanilla 3D Gaussian Splatting (3DGS), which produces "floating" ellipsoids rather than coherent, watertight meshes [S1]. These methods enforce geometric consistency by integrating external priors—such as neural Signed Distance Functions (SDFs), monocular depth estimates, or normal maps—into the training loop, ensuring that Gaussian primitives align with the underlying physical surface [S1]. Unlike siblings that rely on post-hoc mesh extraction or opacity level-sets, these approaches embed surface constraints directly into the optimization objective [S1].

## Shared principles
All methods in this family share the goal of aligning the discrete 3D Gaussian representation with a continuous or metric surface definition. The core mechanism involves adding geometric loss terms to the standard photometric rendering loss. Common strategies include:
1.  **SDF Consistency:** Encouraging the signed distance values of Gaussian centers to be zero, aligning them with an implicit surface [S1].
2.  **Normal Consistency:** Minimizing the angular difference between normals derived from the Gaussians (or rendered normals) and normals predicted by a prior network (e.g., SDF gradient or depth estimator) [S1].
3.  **Depth Alignment:** Ensuring the closest Gaussian to a pixel aligns with rendered or estimated depth values [S1].

## Subtopics

### NeuSG
NeuSG integrates the NeRF-based surface reconstruction technique NeuS into the 3DGS representation to transfer explicit surface properties to the Gaussian primitives [S1]. It solves the problem of vanilla 3DGS producing "floating" ellipsoids by enforcing geometric consistency between the Gaussians and a learned Signed Distance Function (SDF) [S1][S2, unofficial]. Unlike siblings that rely on post-hoc mesh extraction or opacity level-sets, NeuSG integrates surface priors directly into the training loop [S1].

### GSDF
GSDF encourages consistency between a neural Signed Distance Function (SDF) and 3D Gaussian Splatting (3DGS) to enhance geometry reconstruction quality [S1]. It leverages neural SDFs for surface reconstruction to recover more detailed mesh structures, a strategy inspired by NeRF-based methods [S2, unofficial]. It differs from siblings like NeuSG by explicitly transferring surface properties to the 3DGS representation, and from Surface and Geometry Alignment by focusing on the zero-level set consistency of Gaussians themselves.

### [Surface and Geometry Alignment](9267-surface-and-geometry-alignment.md)
[Link to child article](9267-surface-and-geometry-alignment.md)

### GaussianPro
#### definition
- [S1] A method that considers neighboring views' normal direction consistency to produce better reconstruction results.
#### step
- [S1] Introduces a progressive propagation strategy for updating Gaussians, considering normal consistency between neighboring views.
- [S1] Adds plane constraints to guide the growth of 3D Gaussian Splatting.

### Monocular Depth Scale Alignment
#### definition
- [S2, unofficial] A post-optimization step to resolve the up-to-scale problem in 3D scenes reconstructed from multi-view images by aligning the scale of optimized 3D Gaussians with metric-scale monocular depth estimates.
#### pitfall
- [S2, unofficial] Unlike original 3DGS, this method avoids pruning initial COLMAP points to prevent affecting the distribution of optimized Gaussian parameters, ensuring fair comparison with other studies.

### Sparse-view 3D Gaussian Splatting with Depth Priors
#### definition
- [S1] Methods targeting 3D reconstruction from sparse-view inputs by introducing depth inputs estimated by a pre-trained 2D network.
#### step
- [S1] It further removes Gaussians with incorrect depth values and utilizes the [text cuts off].

### 3DGSR
#### definition
- [S1] A method that encourages consistency between Signed Distance Function (SDF) and 3DGS to enhance geometry reconstruction quality.

### DN-Splatter
#### definition
- [S1] A method that utilizes depth and normal priors captured from common devices or predicted from general-purpose networks to enhance the reconstruction quality of 3DGS.

## Background
The discrete nature of 3D Gaussian Splatting often results in geometry that is difficult to extract as a clean mesh, leading to "popping" artifacts and floating structures [S1]. Early attempts to mitigate this included tile-based sorting (StopThePop) [S1]. Subsequent work introduced explicit geometric constraints: GaussianPro used normal consistency and plane constraints [S1]; NeuSG and GSDF incorporated neural SDFs to enforce zero-level set alignment [S1]; and sparse-view methods leveraged pre-trained 2D depth estimators to supervise geometry [S1]. More recently, scale alignment techniques have addressed the metric scale ambiguity inherent in image-only reconstruction pipelines [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2512.20927, fetched 2026-10-26 (unofficial)
