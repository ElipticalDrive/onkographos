# 3D Gaussian Splatting with Normal Information for Mesh Extraction

Also called: GS with Normal Regularization, SDF Gradient Normal Supervision

## Overview
This method augments standard 3D Gaussian Splatting (3DGS) with a regularization term based on the gradients of an estimated Signed Distance Function (SDF) to enforce surface-aligned geometry [S1]. While standard 3DGS relies on photometric losses that often result in imprecise geometry and noisy normals, this approach uses monocular normal estimates to supervise the SDF gradients, enabling high-quality surface mesh extraction via Poisson reconstruction [S1]. It differs from [DN-Splatter](13430-dn-splatter.md) by explicitly deriving SDF gradients from the Gaussian density field rather than using differentiable normal splatting, and from [Gaussian Splatting Photometric Stereo](14017-gaussian-splatting-photometric-stereo.md) by relying on monocular priors rather than multi-view photometric stereo constraints.

## How it works
**Inputs:**
*   A set of calibrated camera poses and images from Structure-from-Motion (SfM) [S1].
*   A sparse point cloud from SfM for initialization [S1].
*   Pretrained monocular normal estimation network (e.g., Omnidata or MiDaS) to generate target normals $\mathbf{n}_p$ for training images [S1].

**Outputs:**
*   Optimized 3D Gaussians (means $\boldsymbol{\mu}$, covariances $\boldsymbol{\Sigma}$, opacities $\alpha$, spherical harmonics coefficients).
*   A surface mesh extracted via Poisson reconstruction from the optimized Gaussians [S1].

**Data Structures:**
*   **Gaussian Primitive:** Defined by mean $\boldsymbol{\mu} \in \mathbb{R}^3$, covariance matrix $\boldsymbol{\Sigma} \in \mathbb{R}^{3 \times 3}$, opacity $\alpha \in [0, 1]$, and spherical harmonics coefficients for color [S1].
*   **Covariance Parameterization:** $\boldsymbol{\Sigma}$ is parameterized by a rotation matrix $\mathbf{R}$ and a scaling matrix $\mathbf{S}$ such that $\boldsymbol{\Sigma} = \mathbf{R} \mathbf{S} \mathbf{S}^T \mathbf{R}^T$ to ensure positive semi-definiteness during optimization [S1].
*   **Density Function:** The scene density at point $\mathbf{p}$ is defined as:
    $$ d(\mathbf{p}) = \sum_{g} \alpha_g \exp \left( -\frac{1}{2} (\mathbf{p} - \boldsymbol{\mu}_g)^T \boldsymbol{\Sigma}_g^{-1} (\mathbf{p} - \boldsymbol{\mu}_g) \right) $$
    where the sum is over all Gaussians $g$ [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize Gaussians using the sparse point cloud obtained from SfM [S1].
2.  **Standard Optimization Loop:**
    *   Project 3D Gaussians to 2D image plane as splats using fast tile-based rasterization [S1].
    *   Render color and depth maps via $\alpha$-blending [S1].
    *   Compute photometric loss ($\mathcal{L}_1$) and D-SSIM loss ($\mathcal{L}_{\text{DSSIM}}$) against input images [S1].
    *   Update Gaussian parameters using gradient descent [S1].
3.  **Adaptive Density Control:** Interleave optimization with density control:
    *   Add Gaussians to empty areas or areas where a single Gaussian covers large parts of the scene [S1].
    *   Prune transparent Gaussians with opacities close to 0 [S1].
4.  **SDF Estimation:** For a point $\mathbf{p}$ visible from a training viewpoint:
    *   Render the depth map $d_{\text{rendered}}$ at the projection of $\mathbf{p}$ [S1].
    *   Estimate the SDF value $\hat{f}(\mathbf{p})$ as the difference between the depth of $\mathbf{p}$ and $d_{\text{rendered}}$ [S1].
5.  **Normal Supervision (Regularization Phase):**
    *   Obtain target monocular normals $\mathbf{n}_p$ from the pretrained network for the training image at pixel $p$ [S1].
    *   Compute the gradient of the estimated SDF, $\nabla \hat{f}_p$, at point $\mathbf{p}$ [S1].
    *   Project $\nabla \hat{f}_p$ into camera space using the world-to-view transform [S1].
    *   Rasterize the projected gradient to obtain $\nabla \hat{f}_p$ at the corresponding pixel position [S1].
    *   Compute the normal regularization term $R(\hat{f})$:
        $$ R(\hat{f}) = \sum_{p \in P} \frac{|\nabla \hat{f}_p \cdot \mathbf{n}_p|}{\|\nabla \hat{f}_p\| \|\mathbf{n}_p\|} $$
        where $P$ is the set of pixel points in the training images [S1].
    *   Compute total loss:
        $$ \mathcal{L} = \mathcal{L}_1 + \lambda_{\text{DSSIM}} \mathcal{L}_{\text{DSSIM}} + \lambda_r R(\hat{f}) $$
        where $\lambda_{\text{DSSIM}} = 0.2$ and $\lambda_r \in [0.1, 0.4]$ [S1].
    *   Apply entropy regularization for a short number of iterations to encourage binary opacities (avoiding semi-transparent Gaussians) [S1].
6.  **Mesh Extraction:** Run Poisson reconstruction on points sampled on the optimized Gaussians to extract the mesh [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_gaussians_with_normal_regularization(cameras, images, sparse_points, normal_net):
    # 1. Initialize Gaussians from sparse points
    gaussians = initialize_gaussians(sparse_points)
    
    # 2. Standard optimization loop
    for iteration in range(max_iterations):
        # Adaptive density control
        gaussians = adaptive_density_control(gaussians, cameras)
        
        # Render current view
        rendered_color, rendered_depth = render_gaussians(gaussians, cameras)
        
        # Photometric losses
        loss_l1 = compute_l1_loss(rendered_color, images)
        loss_dssim = compute_dssim_loss(rendered_color, images)
        
        # Normal regularization (applied during specific iterations or always with weight)
        # Estimate SDF gradient
        sdf_gradients = estimate_sdf_gradients(gaussians, cameras)
        
        # Get monocular normals from pretrained network
        monocular_normals = normal_net(images)
        
        # Project SDF gradients to camera space and rasterize
        rasterized_gradients = rasterize_gradients(sdf_gradients, cameras)
        
        # Compute normal regularization term
        # R(f_hat) = sum(|grad . n| / (||grad|| ||n||))
        normal_loss = compute_cosine_similarity_loss(rasterized_gradients, monocular_normals)
        
        # Total loss
        lambda_dssim = 0.2
        lambda_r = 0.2 # Tuned per scene, typically 0.1-0.4
        total_loss = loss_l1 + lambda_dssim * loss_dssim + lambda_r * normal_loss
        
        # Entropy regularization for binary opacities (short duration)
        if iteration in entropy_regularization_iterations:
            total_loss += entropy_regularization(gaussians.opacities)
            
        # Backpropagate and update
        update_gaussians(gaussians, total_loss)
        
    # 3. Mesh extraction
    mesh = poisson_reconstruction(sample_points_from_gaussians(gaussians))
    return mesh
```

**Parameters:**
*   $\lambda_{\text{DSSIM}}$: Weight for D-SSIM loss, typically 0.2 [S1].
*   $\lambda_r$: Weight for normal regularization, typically between 0.1 and 0.4 [S1].

**Edge Cases and Pitfalls:**
*   **Semi-transparent Gaussians:** Without entropy regularization, Gaussians may remain semi-transparent, degrading mesh quality. Entropy regularization is applied for a short number of iterations to force binary opacities [S1].
*   **Monocular Normal Accuracy:** The method relies on pretrained networks (Omnidata, MiDaS) which may produce inaccurate normals in textureless or ambiguous regions [S1].
*   **SDF Estimation:** The SDF is estimated from depth maps, which may be noisy or incomplete in occluded regions [S1].

## Operating characteristics
*   **Time Complexity:** Similar to standard 3DGS, dominated by rasterization and gradient descent updates. The additional SDF gradient estimation and normal supervision add computational overhead but remain efficient due to tile-based rasterization [S1].
*   **Memory Complexity:** Proportional to the number of Gaussians, similar to standard 3DGS [S1].
*   **Performance:** Scores higher on photorealism metrics (PSNR, SSIM) compared to other mesh-extracting methods like SuGaR and Baked-SDF on datasets such as Mip-NeRF360, Tanks and Temples, and Deep-Blending [S1].
*   **Strengths:** Enables high-quality surface mesh extraction; improves rendering quality in regions with high curvature or fine detail; leverages efficient 3DGS rasterization [S1].
*   **Limitations:** Relies on monocular normal estimation which may be inaccurate; requires careful tuning of $\lambda_r$ per scene [S1].

## Background
This method was proposed by Meenakshi Krishnan, Liam Fowl, and Ramani Duraiswami in 2025 [S1]. It addresses the limitation of standard 3DGS, which relies on photometric losses and often results in imprecise geometry and noisy normals, particularly in regions with high curvature or fine detail [S1]. The method builds on the density function formulation from [8] (SuGaR) but diverges by supervising SDF gradients rather than target density [S1]. It is relevant for downstream applications in video generation, animation, AR-VR, and gaming where surface meshes are required [S1].

## Sources
[S1] 3D Gaussian Splatting with Normal Information for Mesh Extraction and Improved Rendering, https://doi.org/10.1109/icasspw65056.2025.11011170, fetched 2026-10-26
