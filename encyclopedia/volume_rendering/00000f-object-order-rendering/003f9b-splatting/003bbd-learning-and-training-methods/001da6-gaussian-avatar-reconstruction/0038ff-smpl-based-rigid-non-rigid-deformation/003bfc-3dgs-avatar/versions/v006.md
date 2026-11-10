# 3DGS-Avatar


## Overview
3DGS-Avatar reconstructs full-body digital humans by combining 3D Gaussians with rigid and non-rigid transformations derived from an underlying SMPL model [S3, unofficial]. It distinguishes itself from sibling methods by introducing a shallow Multi-Layer Perceptron (MLP) to model Gaussian color, replacing spherical harmonics, and by regularizing the deformation with geometry priors to handle pose-dependent cloth deformation effectively [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Monocular or multi-view video sequences of a human subject; SMPL-X model parameters (pose $\theta$, shape $\beta$, global translation $T$) [S2, S3, unofficial].
*   **Outputs:** A set of 3D Gaussians defined in canonical space, each with position, covariance, opacity, and color parameters, capable of rendering novel poses.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian $i$ stores:
    *   Position $\mathbf{p}_i$ in canonical space.
    *   Rotation $\mathbf{q}_i$ and Scale $\mathbf{s}_i$ (defining covariance $\Sigma_i$).
    *   Opacity $\alpha_i$.
    *   Color representation: Instead of spherical harmonic coefficients, a shallow MLP $\mathcal{M}_{color}$ takes direction $\mathbf{d}$ and pose features as input to predict RGB color [S1].
*   **Deformation Network:** A pose-conditioned MLP $\mathcal{M}_{def}$ that predicts residual offsets $\Delta \mathbf{p}$ based on pose parameters [S1][S3, unofficial].

**Steps:**
1.  **Initialization:** Initialize Gaussians on the surface of the SMPL mesh in the canonical rest pose [S1][S3, unofficial].
2.  **Rigid Deformation:** Apply Linear Blend Skinning (LBS) using SMPL joint rotations to transform canonical positions $\mathbf{p}$ to $\mathbf{p}_{rigid}$ [S1].
3.  **Non-Rigid Deformation:** Predict residual offsets $\Delta \mathbf{p}$ using the deformation network conditioned on pose $\theta$: $\Delta \mathbf{p} = \mathcal{M}_{def}(\theta)$. Compute final position $\mathbf{p}' = \mathbf{p}_{rigid} + \Delta \mathbf{p}$ [S1][S3, unofficial].
4.  **Color Prediction:** For each Gaussian, compute view direction $\mathbf{d}$ in camera space. Pass $\mathbf{d}$ and pose features through the shallow color MLP $\mathcal{M}_{color}$ to obtain RGB color [S1].
5.  **Differentiable Rendering:** Render the deformed Gaussians using standard 2D splatting. Compute photometric loss against input images [S1].
6.  **Regularization:** Apply geometry priors to the deformation field to prevent artifacts, particularly in dynamic cloth regions [S1].

**Formulas:**
*   Rigid Transform: $\mathbf{p}_{rigid} = \text{LBS}(\mathbf{p}, \theta)$
*   Non-Rigid Offset: $\Delta \mathbf{p} = \mathcal{M}_{def}(\theta)$
*   Final Position: $\mathbf{p}' = \mathbf{p}_{rigid} + \Delta \mathbf{p}$
*   Color: $\mathbf{c} = \mathcal{M}_{color}(\mathbf{d}, \theta)$

**Parameters:**
*   Shallow MLP depth: Not given in the sources.
*   Regularization weight for geometry priors: Not given in the sources.

**Edge Cases:**
*   **Monocular Ambiguity:** Without multi-view data, depth ambiguity may cause artifacts in dynamic cloth; addressed by physical-based priors in related methods like GaussianBody, but 3DGS-Avatar relies on geometry priors [S1].
*   **Pose Generalization:** The method is designed to generalize to novel poses, but extreme poses may challenge the deformation network if not sufficiently regularized [S1].

## Subtopics

### Gaussian Avatar
Uses SMPL model with pose features to predict Gaussian parameters via a feed-forward network [S3, unofficial].

### Animatable Gaussians
Uses SMPL-X with explicit pose-conditioned offsets [S2, unofficial].

### UV-based motion parameterization
Maps Gaussians to 2D texture space for appearance capture [S1].

## Operating characteristics
*   **Time Complexity:** Training time is not explicitly given, but the use of a shallow MLP for color and deformation adds computational overhead compared to static 3DGS [S1].
*   **Memory Complexity:** Higher than static 3DGS due to the storage of MLP weights for color and deformation [S1].
*   **Strengths:** Photorealistic rendering with pose-dependent cloth deformation; effective generalization to novel poses [S1].
*   **Limitations:** Requires SMPL-X rig; potential artifacts in dynamic cloth if geometry priors are insufficient [S1].
*   **When to use:** When high-fidelity, animatable full-body avatars with detailed cloth dynamics are needed from video data [S1][S3, unofficial].
*   **When not to use:** For static scenes or when computational resources for MLP inference are limited [S1].

## Background
3DGS-Avatar was introduced to address the limitations of static 3D Gaussian Splatting in dynamic human modeling. It builds on the SMPL-based rigid/non-rigid deformation framework [S3, unofficial]. The method was developed to provide photorealistic rendering with pose-dependent cloth deformation, leveraging a shallow MLP for color and geometry priors for regularization [S1]. It is part of the broader family of Gaussian Avatar Reconstruction methods [S2, S3, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
