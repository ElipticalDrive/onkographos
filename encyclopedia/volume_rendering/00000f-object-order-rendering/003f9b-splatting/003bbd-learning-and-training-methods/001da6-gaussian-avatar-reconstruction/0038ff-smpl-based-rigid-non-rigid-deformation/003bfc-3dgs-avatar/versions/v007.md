# 3DGS-Avatar


## Overview
3DGS-Avatar reconstructs photorealistic, animatable full-body human avatars by anchoring 3D Gaussians to an underlying SMPL model [S3, unofficial]. It distinguishes itself from sibling methods by replacing spherical harmonics with a shallow Multi-Layer Perceptron (MLP) to model per-Gaussian color, enabling the capture of pose-dependent cloth deformation and novel pose generalization [S1].

## How it works
**Inputs:** Monocular or multi-view video sequences of a human subject; SMPL mesh parameters (pose $\theta$, shape $\beta$) fitted to the input frames.
**Outputs:** A set of 3D Gaussians $\{g_i\}$, each defined in a canonical space, with parameters: position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, color $c_i$ (predicted by MLP), and SMPL skinning weights $w_{i,j}$.

**Data Structures:**
*   **Gaussian Attributes:** Stored in flat arrays or structures of arrays (SoA) for GPU access.
    *   $\mathbf{p}_i \in \mathbb{R}^3$: Canonical position.
    *   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix.
    *   $\alpha_i \in \mathbb{R}$: Opacity.
    *   $w_{i,j} \in \mathbb{R}$: Skinning weight for bone $j$ (sums to 1).
    *   MLP Weights $W_{color}$: Parameters for the shallow color network.
*   **SMPL Model:** Pre-loaded mesh topology and joint hierarchy.

**Steps:**
1.  **Initialization:** Initialize Gaussians on the surface of the SMPL mesh in the canonical rest pose [S1][S3, unofficial]. Assign initial skinning weights based on the SMPL vertex weights.
2.  **Deformation:** For a target pose $\theta$:
    a. Compute rigid transformation $\mathbf{p}_{rigid}$ using Linear Blend Skinning (LBS) [S1][S3, unofficial]:
       $$ \mathbf{p}_{rigid} = \sum_{j=1}^{N_{bones}} w_{i,j} \mathbf{T}_j(\theta) \mathbf{p}_i $$
    b. Compute non-rigid residual $\Delta \mathbf{p}$ using a deformation network (MLP) conditioned on pose and canonical position [S3, unofficial].
    c. Final position: $\mathbf{p}'_i = \mathbf{p}_{rigid} + \Delta \mathbf{p}$.
3.  **Covariance Update:** Transform covariance using the Jacobian $J$ of the deformation map [G]:
    $$ \Sigma'_i = J \Sigma_i J^T + \sigma^2 I $$
4.  **Color Prediction:** Instead of spherical harmonics, pass the deformed position, view direction, and pose features through a shallow MLP to predict color $c_i$ [S1].
5.  **Rendering:** Differentiable splatting of deformed Gaussians onto the 2D image plane [S1].
6.  **Optimization:** Minimize loss $\mathcal{L} = \mathcal{L}_{L1} + \lambda_{SSIM}\mathcal{L}_{MS-SSIM} + \lambda_{reg}\mathcal{L}_{reg}$, where $\mathcal{L}_{reg}$ includes geometry priors to regularize deformation [S1].

**Parameters:**
*   MLP depth/width: "Shallow" (specifics not given in sources) [S1].
*   Regularization weights: Not given in sources [S1].

**Edge Cases:**
*   **Degenerate Poses:** Extreme poses may cause skinning weights to become unstable; regularization helps [S1].
*   **Monocular Ambiguity:** Depth ambiguity in monocular video requires strong geometric priors to avoid artifacts [S1].

## Subtopics

### Gaussian Avatar
Uses SMPL with pose features to predict Gaussian parameters via a feed-forward network [S3, unofficial].

### Animatable Gaussians
Uses SMPL-X with explicit pose-conditioned offsets [S2, unofficial].

### UV-based motion parameterization
Maps Gaussians to 2D UV space for appearance modeling [S1].

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ where $N$ is the number of Gaussians. Training is slower due to MLP inference per Gaussian.
*   **Memory Complexity:** Higher than standard 3DGS due to MLP weights and deformation network storage [S1].
*   **Strengths:** Photorealistic rendering of cloth deformation; generalizes to novel poses [S1].
*   **Limitations:** Requires SMPL fitting; MLP inference adds computational overhead compared to SH [S1].

## Background
3DGS-Avatar was introduced to address the limitations of spherical harmonics in capturing complex, pose-dependent appearance changes like cloth wrinkles [S1]. It builds on the SMPL-based rigid/non-rigid deformation framework [S3, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
