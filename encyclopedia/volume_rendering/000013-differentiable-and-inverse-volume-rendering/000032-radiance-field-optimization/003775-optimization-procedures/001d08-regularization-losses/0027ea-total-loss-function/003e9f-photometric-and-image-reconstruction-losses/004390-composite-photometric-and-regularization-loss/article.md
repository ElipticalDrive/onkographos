# Composite Photometric and Regularization Loss

Also called: Combined Loss Function, L_recon + λ_TV L_TV + λ_smooth L_smooth, Model Optimization with Loss Functions, Optimization Loss, Optimization Loss Functions, Training Losses, Training Objective and Regularization

## Overview
This method defines the total training objective for differentiable volume rendering by aggregating a photometric reconstruction term with optional regularization terms to prevent overfitting and ensure physical plausibility [S1, S2, S3, S4, S6, S7]. It builds on [Photometric and Image Reconstruction Losses](9789-photometric-and-image-reconstruction-losses.md) by providing the specific weighted-sum structure $\mathcal{L} = \mathcal{L}_{\text{photo}} + \sum_k \lambda_k \mathcal{L}_{\text{reg}, k}$ used across various radiance field optimization pipelines [S1, S2, S3, S4, S6, S7]. Unlike sibling methods that may focus on visibility weighting or specific single-domain constraints, this composite approach is characterized by its modular inclusion of diverse regularizers (e.g., Total Variation, Eikonal, Compactness) tailored to the specific representation (voxels, Gaussians, SDFs) and task (HDR, dynamic, inverse rendering) [S1, S2, S3, S4][S5, unofficial][S6, S7].

## How it works
The method computes a scalar loss $\mathcal{L}$ by combining a photometric term $\mathcal{L}_{\text{photo}}$ with regularization terms $\mathcal{L}_{\text{reg}, k}$. The specific components and their formulas depend on the rendering representation and task, as detailed in the subtopics below.

### Inputs and Outputs
*   **Inputs:**
    *   Rendered image/patch $\tilde{I}$ or $\mathbf{C}_r$ (from the differentiable renderer).
    *   Ground-truth image/patch $I$ or $\mathbf{C}_{gt}$.
    *   Scene parameters: Voxel grid values (opacity $\sigma$, SH coefficients), Gaussian parameters (position, scale $\mathbf{s}$, opacity $\alpha$), SDF values $S$, material parameters.
    *   Camera parameters and ray set $\mathcal{R}$.
    *   Optional: Saturation masks $M_i$, CRF functions $g_i$, flow fields $\mathbf{V}_{flow}$.
*   **Outputs:**
    *   Scalar loss value $\mathcal{L}$ for backpropagation.

### Data Structures
*   **Ray Set $\mathcal{R}$:** Collection of rays sampled for the current batch.
*   **Voxel Grid:** 3D array storing density $\sigma$ and color features (e.g., SH coefficients).
*   **Gaussian Primitives:** List of objects with attributes: position $\mathbf{p}$, scale $\mathbf{s} \in \mathbb{R}^3$, opacity $\alpha$, rotation $\mathbf{q}$, color coefficients.
*   **SDF Field:** Neural network or grid representing signed distance function $S(\mathbf{x})$.

### General Form
$$ \mathcal{L} = \mathcal{L}_{\text{photo}} + \sum_k \lambda_k \mathcal{L}_{\text{reg}, k} $$

### Photometric Terms (Examples)
The photometric term varies by source:
1.  **LDR Reconstruction with Saturation Masking [S1]:**
    $$ \mathcal{L}_{\text{recon}} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} M_i(\mathbf{r}) \| I_i(\Pi_i(\mathbf{r})) - \mathcal{T}(\hat{C}(\mathbf{r})) \|_2^2 $$
    where $\Pi_i$ is camera projection, $\mathcal{T}$ is tone mapping, and $M_i$ masks saturated pixels.
2.  **Dual-Grid Photometric Loss [S2]:**
    $$ \mathcal{L}_{\text{photo}} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \| \mathbf{C}_{\text{Inp}}(\mathbf{r}) - \mathbf{C}_{\text{gt}}(\mathbf{r}) \|_2^2 + \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \| \mathbf{C}_{\text{Up}}(\mathbf{r}) - \mathbf{C}_{\text{gt}}(\mathbf{r}) \|_2^2 $$
    where $\mathbf{C}_{\text{Inp}}$ is rendered from an inpainted grid and $\mathbf{C}_{\text{Up}}$ from an up-sampled grid.
3.  **L1 + D-SSIM [S3]:**
    $$ \mathcal{L}_{\text{photo}} = (1 - \lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{\text{D-SSIM}} $$
4.  **Perceptual Loss [S4]:**
    $$ \mathcal{L}_{\text{rgb}} = \| \tilde{I}_P - I_P \|_2^2 + \| F_{\text{vgg}} (\tilde{I}_P) - F_{\text{vgg}} (I_P) \|_2^2 $$
5.  **Multi-scale L2 + SSIM [S6]:**
    $$ \mathcal{L}_{\text{photo}} = L_2(\text{pyramid}(\hat{I}), \text{pyramid}(I)) + (1 - \text{SSIM}(\hat{I}, I)) $$
6.  **Compact 3DGS [S7]:**
    $$ \mathcal{L}_{\text{photo}} = (1-\lambda_D)L_1 + \lambda_D L_{D-SSIM} $$

### Regularization Terms (Examples)
1.  **Total Variation (TV) [S1, S2]:**
    $$ \mathcal{L}_{\text{TV}}(V) = \frac{1}{|\mathcal{V}|} \sum_{\mathbf{v} \in \mathcal{V}, d \in [D]} \sqrt{\Delta_x^2(\mathbf{v}, d) + \Delta_y^2(\mathbf{v}, d) + \Delta_z^2(\mathbf{v}, d) + \epsilon} $$
    Applied to opacity $\sigma$, SH coefficients, flow fields, or depth $D$.
2.  **Smoothness Loss (CRF) [S1]:**
    $$ \mathcal{L}_{\text{smooth}} = \sum_{i=1}^N \sum_{e \in [0,1]} g_i''(e)^2 $$
3.  **Opacity/Background Entropy [S2]:**
    $$ \mathcal{L}_{\text{bg}} \quad \text{(Background entropy loss)} $$
4.  **Static Motion Prior [S2]:**
    $$ \mathcal{L}_{\text{flow}} = \| \mathbf{V}_{\text{flow}}^{\text{Can}} \|_1 $$
5.  **Inpainting Difference [S2]:**
    $$ \mathcal{L}_{\text{vdiff}} = \| \mathbf{V}_{\text{R}_{\text{Inp}}}^t - \mathbf{V}_{\text{R}_{\text{Up}}}^t \|_1 $$
6.  **Eikonal Loss [S6]:**
    $$ \mathcal{L}_{\text{eikonal}} = \| \nabla_x S - 1 \|_2^2 $$
7.  **Roughness Range Loss [S6]:**
    Encourages estimated roughness to stay below 0.5. Formula not explicitly provided, referred to as Eq. 13 in source [S6].
8.  **Triangle Size Regularization [S3]:**
    $$ \mathcal{L}_s = 2 \| (\mathbf{v}_1 - \mathbf{v}_0) \times (\mathbf{v}_2 - \mathbf{v}_0) \|^{-1} $$
    where $\mathbf{v}_i$ are vertices sampled from the unit sphere scaled by distance $d$ and centered at point $\mathbf{q}$: $\mathbf{v}_i = \mathbf{q} + k \cdot d \cdot \mathbf{u}_i$.
9.  **Gaussian Volume Regularization [S7]:**
    $$ \mathcal{L}_{\text{vol}} = \frac{1}{|\mathcal{G}|} \sum_i \prod_{j=1}^3 s_{i,j} $$
10. **Gaussian Opacity Regularization [S7]:**
    $$ \mathcal{L}_{\text{op}} = \frac{1}{|\mathcal{G}|} \sum_i \alpha_i $$
11. **Scaling Regularization [S5, unofficial]:**
    $$ \mathcal{L}_{\text{scaling}}(i) = \begin{cases} |\hat{s}_i|, & \hat{s}_i > \max(T_s, T_r \tilde{s}_i) \\ 0, & \text{otherwise} \end{cases} $$
    where $\hat{s}_i$ is max scaling, $\tilde{s}_i$ is min scaling, $T_s$ is length threshold, $T_r$ is thinness ratio.

### Parameters
*   **Weights:** $\lambda_k$ are hyperparameters balancing the terms. Typical values:
    *   $\lambda_{\text{TV}, \sigma} = 5 \cdot 10^{-4}$, $\lambda_{\text{TV}, \text{SH}} = 1 \cdot 10^{-2}$, $\lambda_{\text{smooth}} = 1 \cdot 10^{-3}$ [S1].
    *   $\lambda_D = 0.2$ (D-SSIM weight) [S7].
    *   $\lambda_s = \lambda_o = 0.01$ (Compactness weights) [S7].
    *   $w_1 \sim w_7$ balance components in dynamic scene loss [S2].
    *   $\beta_1, \beta_2, \beta_3, \beta_4$ balance opacity, distortion, normal, and size losses [S3].
    *   $\lambda_1, \lambda_2$ balance Eikonal and roughness losses [S6].
*   **Thresholds:** $T_s$ (length threshold), $T_r$ (thinness ratio) [S5, unofficial].

### Edge Cases and Pitfalls
*   **Saturation Masking:** In HDR pipelines, failing to mask saturated pixels leads to incorrect gradient signals [S1].
*   **Background Mixing:** For monocular setups, generating a random background color every iteration helps provide silhouette cues [S5, unofficial].
*   **Scaling Regularization:** Applied only when a Gaussian is both long ($\hat{s}_i > T_s$) and thin ($\hat{s}_i > T_r \tilde{s}_i$) [S5, unofficial].

## Subtopics

### [MPF Training Objective](10247-mpf-training-objective.md)
The MPF Training Objective is a composite loss function designed to optimize the parameters of a Metric-Phase Field (MPF) network for multi-view image reconstruction. It combines standard photometric supervision with geometric regularizers that exploit rendering weights as soft indicators for near-surface and free-space regions, addressing the lack of ground-truth 3D geometry in this setting. Unlike the original MPF formulation which relies on explicit point cloud supervision, this method adapts the metric-phase decomposition constraints to be implicit and differentiable through the rendering process.

### Scaling and Background Mixing Regularization
A variant for monocular 3DGS training that prevents Gaussians from growing long and thin. It introduces a scaling regularization term $\mathcal{L}_{\text{scaling}}$ and uses background mixing (random background color mixed with rendered and ground truth images) to provide silhouette cues [S5, unofficial]. The scaling loss penalizes the max scaling $\hat{s}_i$ if it exceeds both a length threshold $T_s$ and a thinness ratio $T_r$ times the min scaling $\tilde{s}_i$ [S5, unofficial].

### Inverse Rendering Loss Function
A composite loss function used to optimize neural SDF and material parameters from multi-view photometric images. It combines image fidelity (L2 on Gaussian pyramids), structural similarity (1 - SSIM), SDF validity (Eikonal loss), and material constraints (Roughness Range Loss) [S6]. The Eikonal loss enforces $\|\nabla_x S - 1\|_2^2 = 0$, and the roughness loss encourages estimated roughness to stay below 0.5 [S6].

### Compactness-Regularized Objective
An optimization objective that augments the standard 3DGS photometric loss with two lightweight compactness terms: a volume term (product of scale axes) and an opacity term (L1 on opacity). The volume term $\frac{1}{|\mathcal{G}|} \sum_i \prod_{j=1}^3 s_{i,j}$ penalizes oversized primitives more sharply than their sum, and the opacity term $\frac{1}{|\mathcal{G}|} \sum_i \alpha_i$ drives redundant Gaussians below the prune threshold [S7]. This replaces the L1 penalty on scale axes used in 3D-GS-MCMC [S7].

## Operating characteristics
*   **Time Complexity:** Depends on the number of rays $|\mathcal{R}|$, voxels $|\mathcal{V}|$, or Gaussians $|\mathcal{G}|$. Regularization terms like TV and Eikonal add overhead proportional to the number of grid points or sampled points.
*   **Memory Complexity:** Requires storing gradients for all parameters (voxel values, Gaussian attributes, SDF network weights).
*   **Strengths:** Flexible framework allowing customization of regularizers for specific artifacts (e.g., noise, overfitting, unnatural shapes).
*   **Limitations:** Requires careful tuning of multiple hyperparameters ($\lambda_k$). Some regularizers (e.g., TV on high-res grids) can be computationally expensive.
*   **When to Use:** When standard photometric loss leads to overfitting, noise, or physically implausible structures.
*   **When Not to Use:** When computational resources are extremely limited and hyperparameter tuning is infeasible.

## Background
The concept of combining photometric and regularization losses is fundamental to differentiable rendering and inverse graphics. Early works like NeRF [S4] utilized perceptual and density concentration losses. Subsequent works adapted these principles to specific domains: HDR rendering [S1], dynamic scenes [S2], triangle splatting [S3], inverse rendering with SDFs [S6], and compact 3D Gaussian Splatting [S7]. The specific formulations vary based on the representation (voxels, Gaussians, SDFs) and the desired properties (smoothness, compactness, physical plausibility).

## Sources
[S1] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
[S2] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
[S3] Triangle Splatting for Real-Time Radiance Field Rendering, https://doi.org/10.1109/3DV69130.2026.00123, fetched 2026-10-27
[S4] Learning Neural Volumetric Representations of Dynamic Humans in Minutes, https://doi.org/10.1109/cvpr52729.2023.00846, fetched 2026-10-30
[S5] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
[S6] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26
[S7] DART-3DGS: Distance-Aware Replication and Age-Gated Densification for 3D Gaussian Splatting, https://europepmc.org/article/PMC/PMC13705904, fetched 2026-10-27
