# 3DGS-Avatar
## Overview
3DGS-Avatar reconstructs photorealistic full-body avatars by combining 3D Gaussians with both non-rigid and rigid transformations of an underlying SMPL model [S3, unofficial]. It distinguishes itself from sibling methods by introducing a shallow Multi-Layer Perceptron (MLP) to model Gaussian color, replacing spherical harmonics, and regularizing deformation with geometry priors to handle pose-dependent cloth deformation effectively [S1].
## How it works
**Inputs and Outputs:**
*   **Inputs:** Monocular or multi-view video of a human subject; SMPL-X model parameters (pose $\beta$, shape $\theta$, global translation) estimated via fitting [S2, unofficial][S3, unofficial].
*   **Outputs:** A set of 3D Gaussians $\{G_i\}$, each with position, covariance, opacity, and color parameters, deformable according to SMPL pose.

**Data Structures:**
*   **Gaussian Set:** Each Gaussian $i$ stores:
    *   Position $\mathbf{p}_i$ in canonical space.
    *   Rotation $\mathbf{q}_i$ and Scale $\mathbf{s}_i$ (defining covariance $\Sigma_i$).
    *   Opacity $\alpha_i$.
    *   Color representation: Parameters for a shallow MLP $C_i(\cdot)$ instead of spherical harmonic coefficients [S1].
*   **Deformation Network:** A per-Gaussian MLP $\mathcal{D}_i(\cdot)$ that predicts deformation offsets based on SMPL pose parameters [S2, unofficial].

**Steps:**
1.  **Initialization:** Initialize Gaussians on the SMPL mesh surface in canonical space [S2, unofficial].
2.  **Deformation Prediction:** For a target pose $\theta$, compute the rigid transformation $T_{rigid}(\theta)$ from SMPL. For each Gaussian $i$, predict a non-rigid offset $\Delta \mathbf{p}_i$ using the per-Gaussian deformation MLP: $\Delta \mathbf{p}_i = \mathcal{D}_i(\theta)$ [S2, unofficial]. The final position is $\mathbf{p}_i' = T_{rigid}(\theta) \mathbf{p}_i + \Delta \mathbf{p}_i$.
3.  **Color Prediction:** Compute view-dependent color using the shallow MLP: $\mathbf{c}_i = C_i(\mathbf{v}, \theta)$, where $\mathbf{v}$ is the view direction [S1].
4.  **Rendering:** Rasterize the transformed Gaussians using standard differentiable splatting [S1].
5.  **Optimization:** Minimize photometric loss $\mathcal{L}_{photo}$ between rendered and ground-truth images. Regularize deformation with geometry priors to prevent artifacts [S1].

**Formulas:**
*   **Covariance:** $\Sigma_i = R_i S_i S_i^T R_i^T$, where $R_i$ is rotation matrix from $\mathbf{q}_i$ and $S_i$ is diagonal scale matrix from $\mathbf{s}_i$.
*   **Deformation:** $\mathbf{p}_i^{deformed} = \mathbf{T}_{SMPL}(\theta) \mathbf{p}_i^{canonical} + \mathcal{D}_i(\theta)$ [S2, unofficial].
*   **Color:** $\mathbf{c}_i = \text{MLP}_{color}(\mathbf{p}_i, \mathbf{v}, \theta)$ [S1].

**Parameters:**
*   MLP depth/width: Shallow (specifics not given in sources) [S1].
*   Regularization weight: Not given in sources [S1].

**Edge Cases:**
*   **Monocular Ambiguity:** May cause artifacts in dynamic cloth without physical-based priors (addressed by GaussianBody [S1]).
*   **Pose Generalization:** Requires effective regularization to generalize to novel poses [S1].

## Subtopics

### Gaussian Avatar
Uses SMPL model with pose features to predict Gaussian parameters via a feed-forward network [S3, unofficial].

### Animatable Gaussians
Uses SMPL-X with explicit pose-conditioned offsets for deformation [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Real-time rendering is enabled by the efficient splatting pipeline [S1]. Training time depends on MLP optimization [S1].
*   **Memory Complexity:** Higher than standard 3DGS due to per-Gaussian MLP parameters [S1].
*   **Strengths:** Photorealistic rendering, handles pose-dependent cloth deformation, generalizes to novel poses [S1].
*   **Limitations:** Requires SMPL-X fitting; potential artifacts in monocular video without additional priors [S1].
*   **When to use:** Full-body avatar reconstruction with high-fidelity cloth details.
*   **When not to use:** Real-time applications with strict memory constraints due to MLP overhead [S1].

## Background
3DGS-Avatar was introduced to address the limitations of standard 3DGS in handling dynamic human avatars, specifically cloth deformation [S1]. It builds on the SMPL-X rig and per-Gaussian deformation MLPs [S2, unofficial]. It is part of the broader family of Gaussian Avatar Reconstruction methods [S3, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
