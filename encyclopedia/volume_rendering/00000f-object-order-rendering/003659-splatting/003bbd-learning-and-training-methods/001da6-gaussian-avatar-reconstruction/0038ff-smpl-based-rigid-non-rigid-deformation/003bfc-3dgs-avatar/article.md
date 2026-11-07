# 3DGS-Avatar


## Overview
3DGS-Avatar reconstructs photorealistic, animatable full-body human avatars by combining 3D Gaussian primitives with an underlying SMPL model [S3, unofficial]. It addresses the challenge of rendering pose-dependent cloth deformation and fine geometric details that standard spherical harmonics (SH) often blur [S1]. It distinguishes itself from sibling methods like [Gaussian Avatar](1479-gaussian-avatar.md) by employing a shallow Multi-Layer Perceptron (MLP) for color modeling and explicit geometry priors for deformation regularization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Monocular or multi-view video of a human subject; SMPL mesh parameters (pose $\beta$, shape $\theta$) estimated via fitting algorithms.
*   **Output:** A set of 3D Gaussians with parameters optimized to render novel views and poses.

**Data Structures:**
*   **Gaussian Primitives:** Each Gaussian $i$ is defined by position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), opacity $\alpha_i$, and color representation.
*   **Color Representation:** Unlike standard 3DGS which uses SH coefficients, 3DGS-Avatar uses a shallow MLP $\mathcal{M}_{color}$ to predict color $\mathbf{c}_i$ [S1].
*   **Deformation:** Gaussians are associated with the SMPL mesh. Their positions and orientations are functions of the SMPL pose parameters [S3, unofficial].

**Steps:**
1.  **Initialization:** Initialize Gaussians on the SMPL mesh surface [S3, unofficial].
2.  **Deformation:** Apply rigid and non-rigid transformations based on the SMPL pose $\theta$ to move Gaussians from canonical space to the target pose [S3, unofficial].
3.  **Color Prediction:** For each Gaussian, compute its color using the shallow MLP $\mathcal{M}_{color}$, conditioned on the Gaussian's position and potentially pose features [S1].
4.  **Rendering:** Render the deformed Gaussians using differentiable splatting [S1][S3, unofficial].
5.  **Optimization:** Minimize photometric loss between rendered and input images. Regularize the deformation using geometry priors to prevent artifacts [S1].

**Formulas:**
*   **Color:** $\mathbf{c}_i = \mathcal{M}_{color}(\mathbf{p}_i, \dots)$ [S1].
*   **Deformation:** $\mathbf{p}_i^{posed} = \mathcal{T}_{SMPL}(\mathbf{p}_i^{canonical}, \theta)$ [S3, unofficial].

**Parameters:**
*   **MLP Depth:** Shallow (specific depth not given in sources) [S1].
*   **SMPL Model:** SMPL-X or SMPL [S2, S3, unofficial].

**Edge Cases:**
*   **Monocular Ambiguity:** Requires physical-based priors or geometry regularization to avoid dynamic cloth artifacts [S1].
*   **Novel Poses:** Generalization to unseen poses depends on the robustness of the deformation MLP and priors [S1].

## Subtopics

### Gaussian Avatar
Uses SMPL prior with pose features to predict Gaussian parameters via a feed-forward network [S3, unofficial].

### Animatable Gaussians
Uses SMPL-X with explicit pose-conditioned offsets [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Real-time rendering is enabled by the Gaussian splatting rasterization [S1]. Training time depends on MLP optimization and densification.
*   **Memory Complexity:** Higher than standard 3DGS due to MLP weights per Gaussian or shared MLP parameters [S1].
*   **Strengths:** Photorealistic rendering of cloth details; effective generalization to novel poses [S1].
*   **Limitations:** Requires careful regularization to avoid artifacts in dynamic cloth from monocular video [S1].

## Background
3DGS-Avatar was introduced to improve the reconstruction of rich human details, such as clothing, by replacing spherical harmonics with a shallow MLP for color modeling and incorporating geometry priors for deformation [S1]. It builds on the SMPL-based rigid/non-rigid deformation framework [S3, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
