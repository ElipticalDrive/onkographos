# Normal Supervision and Reconstruction

Also called: Normal-guided Gaussian Splatting, Normal Regularization in 3DGS

## Overview
Normal Supervision and Reconstruction refers to a family of techniques that enhance 3D Gaussian Splatting (3DGS) by incorporating surface normal constraints into the optimization process. These methods solve the problem of ambiguous or noisy geometry inherent in standard volumetric rendering, where photometric losses alone often fail to recover precise surface details or coherent normals. What sets this family apart from its siblings, such as [Gaussian Frosting Layer Creation](12579-gaussian-frosting-layer-creation.md) or [Ambiguous Gaussians Removal](19220-ambiguous-gaussians-removal.md), is the explicit use of normal alignment—either against monocular estimates, depth gradients, or photometric stereo cues—to enforce surface-aligned geometry rather than just density fitting.

## Shared principles
The core mechanism involves augmenting the standard 3DGS rendering pipeline with a normal regularization loss term. This requires:
1.  **Normal Representation:** Extending the Gaussian primitive to include or infer a normal vector $\mathbf{n}_i$. In standard 3DGS, normals are not explicit parameters; they must be derived (e.g., from SDF gradients or depth) or explicitly optimized.
2.  **Normal Rendering:** Computing a rendered normal map $\hat{\mathbf{n}}_{uv}$ for a pixel $(u,v)$ by alpha-blending the normals of contributing Gaussians in depth order:
    $$ \hat{\mathbf{n}}_{uv} = \sum_{i}^{N} \mathbf{n}_{i} \alpha_{i} \prod_{j=1}^{i-1} (1-\alpha_{j}) $$
    where $\mathbf{n}_i$ is the normal of the $i$-th Gaussian, $\alpha_i$ is its opacity, and the product term represents the transparency of preceding Gaussians [S2].
3.  **Supervision Signal:** Comparing the rendered normal $\hat{\mathbf{n}}_{uv}$ against a target normal $\mathbf{n}_{uv}$. The target may come from:
    *   Monocular normal estimation networks (e.g., Omnidata, MiDaS) [S1][S2].
    *   Gradients of an estimated Signed Distance Function (SDF) or depth map [S1].
    *   Photometric stereo constraints under known lighting [S1].
4.  **Loss Function:** A cosine distance or similarity loss is added to the total optimization loss:
    $$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photometric}} + \lambda_{\text{normal}} \mathcal{L}_{\text{normal}} $$
    where $\mathcal{L}_{\text{normal}}$ measures the alignment between $\hat{\mathbf{n}}_{uv}$ and $\mathbf{n}_{uv}$ [S1][S2].

## Subtopics

### [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md)
This method extends 3DGS by introducing a regularization term based on the gradients of an estimated Signed Distance Function (SDF) to enforce surface-aligned geometry. While standard GS relies on photometric losses that often result in imprecise geometry and noisy normals, this approach enables high-quality surface mesh extraction and improved rendering fidelity, particularly in regions with high curvature or fine detail. It distinguishes itself from sibling methods like [Gaussian Frosting Layer Creation](12579-gaussian-frosting-layer-creation.md) and [Dual-Domain Deformation Model](12804-dual-domain-deformation-model.md) by explicitly supervising the SDF gradient direction against monocular normal estimates to ensure the Gaussians represent a coherent surface rather than a volumetric density field.

### [Gaussian Splatting Photometric Stereo](14017-gaussian-splatting-photometric-stereo.md)
Gaussian Splatting Photometric Stereo (GS-PS) is a calibrated photometric stereo method that estimates surface normals and albedo by optimizing 3D Gaussian Splatting parameters to minimize a combined photometric and normal alignment loss. It differs from deep learning-based photometric stereo by optimizing an explicit physical image formation model rather than neural network weights, and from inverse rendering methods using small networks by using Gaussian parameters with a direct geometric interpretation.

### Normal Reconstruction via Depth Gradient
*   **Definition:** A strategy to estimate scene normals in 3DGS by improving depth estimation and using the depth gradient to derive pseudo normals, which guide the optimization of normals stored in 3D Gaussians [S1].
*   **Formula:** Pseudo normals $\hat{\mathbf{n}}_{\hat{D}}$ are derived from the depth map gradient: $\hat{\mathbf{n}}_{\hat{D}} = \nabla_{uv} \hat{D}$ [S1].

### Normal Vector Learning via 2D Lifting
*   **Definition:** Learning 3D normal vectors by rendering normals from the 3D Gaussian field and comparing them with predictions from an off-the-shelf 2D normal estimation network (Omnidata) [S2].
*   **Step:** Render the normal vector $\hat{\mathbf{n}}_{uv}$ for each pixel $(u,v)$ by alpha-blending the normal vectors $\mathbf{n}_i$ of the Gaussians, weighted by their opacity $\alpha_i$ and the transparency of preceding Gaussians in depth order [S2].
*   **Formula:** The normal loss is defined as $\mathcal{L}_{\text{n}} = \sum_{(u,v)} (1 - \langle \hat{\mathbf{n}}_{uv} \cdot \mathbf{n}_{uv} \rangle)$, where $\hat{\mathbf{n}}_{uv}$ is the rendered normal at pixel $(u,v)$, and $\mathbf{n}_{uv}$ is the predicted normal from the 2D network. The rendered normal is computed as $\hat{\mathbf{n}}_{uv} = \sum_{i} \mathbf{n}_{i} \alpha_{i} \prod_{j=1}^{i-1} (1-\alpha_{j})$ [S2].

## Background
The integration of normal supervision into 3D Gaussian Splatting emerged as a solution to the "floating geometry" and lack of explicit surface representation in standard 3DGS. Standard 3DGS optimizes for photometric consistency, which can lead to semi-transparent Gaussians and ambiguous depth, making mesh extraction difficult [S1]. Early works like [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-mesh-extra.md) introduced SDF-based normal regularization to enforce surface coherence [S1]. Concurrently, methods like [Planar Gaussian Splatting](12804-dual-domain-deformation-model.md) and [Gaussian Splatting Photometric Stereo](14017-gaussian-splatting-photometric-stereo.md) explored explicit normal parameterization and photometric constraints to improve geometric accuracy and enable inverse rendering tasks [S1][S2]. These approaches leverage pretrained monocular normal estimators (e.g., Omnidata [S1][S2]) or depth gradients [S1] as weak supervision signals to guide the 3D structure.

## Sources
[S1] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
[S2] Planar Gaussian Splatting, https://doi.org/10.1109/wacv61041.2025.00863, fetched 2026-10-26
