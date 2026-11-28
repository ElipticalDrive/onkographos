# Normal Supervision and Reconstruction


## Overview
Normal Supervision and Reconstruction refers to a family of techniques that enhance 3D Gaussian Splatting (3DGS) by incorporating surface normal constraints into the optimization process [G]. These methods solve the problem of ambiguous or noisy geometry inherent in standard volumetric rendering, where photometric losses alone often fail to recover precise surface details or coherent normals [G]. What sets this family apart from its siblings, such as [Gaussian Frosting Layer Creation](12579-gaussian-frosting-layer-creation.md) or [Ambiguous Gaussians Removal](19220-ambiguous-gaussians-removal.md), is the explicit use of normal alignment—either against monocular estimates, depth gradients, or photometric stereo cues—to enforce surface-aligned geometry rather than just density fitting [S1, S2, S3].

## Shared principles
The core mechanism involves augmenting the standard 3DGS rendering pipeline with a normal regularization loss term. This requires:
1. **Normal Representation:** Extending the Gaussian primitive to include or infer a normal vector $\mathbf{n}_i$. In standard 3DGS, normals are not explicit parameters; they must be derived (e.g., from depth gradients) or explicitly optimized [S1, S2, S3].
2. **Normal Rendering:** Computing a rendered normal map $\hat{\mathbf{n}}_{uv}$ for a pixel $(u,v)$ by alpha-blending the normals of contributing Gaussians in depth order:
   $$ \hat{\mathbf{n}}_{uv} = \sum_{i}^{N} \mathbf{n}_{i} \alpha_{i} \prod_{j=1}^{i-1} (1-\alpha_{j}) $$
   where $\mathbf{n}_i$ is the normal of the $i$-th Gaussian, $\alpha_i$ is its opacity, and the product term represents the transparency of preceding Gaussians [S2].
3. **Supervision Signal:** Comparing the rendered normal $\hat{\mathbf{n}}_{uv}$ against a target normal $\mathbf{n}_{uv}$. Targets are derived from monocular normal estimation networks (e.g., Omnidata, MiDaS) [S1, S2], gradients of estimated depth maps [S1], or D-Normal regularization from back-projected depth [S3].
4. **Loss Function:** A cosine distance or similarity loss is added to the total optimization loss:
   $$ \mathcal{L}_{\text{total}} = \mathcal{L}_{\text{photometric}} + \lambda_{\text{normal}} \mathcal{L}_{\text{normal}} $$
   where $\mathcal{L}_{\text{normal}}$ measures alignment between $\hat{\mathbf{n}}_{uv}$ and $\mathbf{n}_{uv}$ [S1, S2, S3].

## Subtopics

### [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md)
This method enhances 3DGS by incorporating normal supervision derived from the gradients of an estimated depth map to improve geometric accuracy and enable high-quality mesh extraction. While standard 3DGS relies solely on photometric losses, which often result in imprecise geometry and noisy normals, this approach uses monocular normal estimates to regularize depth gradients, facilitating the reconstruction of surface-aligned Gaussians suitable for Poisson meshing. It distinguishes itself from sibling methods by explicitly linking the depth gradient direction to monocular normal priors rather than relying on depth consistency or photometric stereo alone.

### [Gaussian Splatting Photometric Stereo](14017-gaussian-splatting-photometric-stereo.md)
Gaussian Splatting Photometric Stereo (GS-PS) is a calibrated photometric stereo method that represents the scene exclusively using 3D Gaussian primitives to estimate surface normals and albedo. It solves the problem of recovering high-frequency surface geometry from multi-view images under varying lighting by optimizing Gaussian parameters to minimize a combined photometric and normal alignment loss. Unlike deep neural network methods that predict normals directly from image features, GS-PS optimizes an explicit physical image formation model, making it more explainable and providing geometrically interpretable parameters. It builds on this family by integrating normal supervision directly into the Gaussian splatting optimization loop.

### Normal Reconstruction via Depth Gradient
*   **Definition:** A strategy to estimate scene normals in 3DGS by improving depth estimation and using the depth gradient to derive pseudo normals, which guide the optimization of normals stored in 3D Gaussians [S1].
*   **Formula:** Pseudo normals $\hat{\mathbf{n}}_{\hat{D}}$ are derived from the depth map gradient: $\hat{\mathbf{n}}_{\hat{D}} = \nabla_{uv} \hat{D}$ [S1].

### Normal Vector Learning via 2D Lifting
*   **Definition:** Learning 3D normal vectors by rendering normals from the 3D Gaussian field and comparing them with predictions from an off-the-shelf 2D normal estimation network (Omnidata) [S2].
*   **Step:** Render the normal vector $\hat{\mathbf{n}}_{uv}$ for each pixel $(u,v)$ by alpha-blending the normal vectors $\mathbf{n}_i$ of the Gaussians, weighted by their opacity $\alpha_i$ and the transparency of preceding Gaussians in depth order [S2].
*   **Formula:** The normal loss is defined as $\mathcal{L}_{\text{n}} = \sum_{(u,v)} (1 - \langle \hat{\mathbf{n}}_{uv} \cdot \mathbf{n}_{uv} \rangle)$, where $\hat{\mathbf{n}}_{uv}$ is the rendered normal at pixel $(u,v)$, and $\mathbf{n}_{uv}$ is the predicted normal from the 2D network. The rendered normal is computed as $\hat{\mathbf{n}}_{uv} = \sum_{i} \mathbf{n}_{i} \alpha_{i} \prod_{j=1}^{i-1} (1-\alpha_{j})$ [S2].

### View-Consistent D-Normal Regularization
*   **Definition:** A regularizer that uses a normal prior $\mathbf{N}$ from a pretrained monocular deep neural network to supervise the rendered normal map $\hat{\mathbf{N}}$ with L1 and cosine losses [S3].
*   **Step:** Render the depth map $\hat{D}$ by weighted summing the intersection depths $d_i$. Convert the rendered depth $\hat{D}$ to a D-Normal $\bar{\mathbf{N}}_d$ by back-projecting the depth map into point clouds with the camera intrinsic matrix. Compute the confidence term $w$ as the cosine distance between the rendered normal $\hat{\mathbf{N}}_d$ and predicted normal $\mathbf{N}$ to emphasize high certainty areas [S3].
*   **Formula:** $\mathcal{L}_n = \|\hat{\mathbf{N}} - \mathbf{N}\|_1 + (1 - \hat{\mathbf{N}} \cdot \mathbf{N})$, where $\hat{\mathbf{N}} = (\sum_{i \in M} \mathbf{n}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)) / (\sum_{i \in M} \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j))$ [S3]. $\bar{\mathbf{N}}_d(n, p) = (\nabla_v d(n, p) \times \nabla_h d(n, p)) / |\nabla_v d(n, p) \times \nabla_h d(n, p)|$, where $\nabla_v$ and $\nabla_h$ are horizontal and vertical finite differences from neighboring points [S3]. $\mathcal{L}_{dn} = w * (\|\bar{\mathbf{N}}_d - \mathbf{N}\|_1 + (1 - \bar{\mathbf{N}}_d \cdot \mathbf{N}))$ [S3]. $\mathcal{L}_{total} = \mathcal{L}_{RGB} + \lambda_1 \mathcal{L}_s + \lambda_2 \mathcal{L}_n + \lambda_3 \mathcal{L}_{dn}$, with $\lambda_1, \lambda_2, \lambda_3$ balancing the components [S3].

## Background
The integration of normal supervision into 3D Gaussian Splatting emerged as a solution to the "floating geometry" and lack of explicit surface representation in standard 3DGS [G]. Early works like [3D Gaussian Splatting with Normal Information for Mesh Extraction](12069-3d-gaussian-splatting-with-normal-information-for-mesh-extra.md) introduced depth-gradient-based normal regularization to enforce surface coherence [S1]. Concurrently, methods like [Gaussian Splatting Photometric Stereo](14017-gaussian-splatting-photometric-stereo.md) explored explicit normal parameterization and photometric constraints to improve geometric accuracy and enable inverse rendering tasks [S1, S2]. These approaches leverage pretrained monocular normal estimators (e.g., Omnidata [S1, S2]) or depth gradients [S1] as weak supervision signals to guide the 3D structure.

## Sources
[S1] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
[S2] Planar Gaussian Splatting, https://doi.org/10.1109/wacv61041.2025.00863, fetched 2026-10-26
[S3] VCR-GauS: View Consistent Depth-Normal Regularizer for Gaussian Surface Reconstruction, https://doi.org/10.52202/079017-4434, fetched 2026-10-29
