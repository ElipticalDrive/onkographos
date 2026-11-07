# Composite Photometric and Regularization Loss

Also called: Combined Loss Function, Model Optimization with Loss Functions, Optimization Loss, Training Losses

## Overview
This method defines the total objective function for optimizing differentiable volume rendering or splatting models by combining a photometric reconstruction term with multiple regularization priors. It addresses the ill-posed nature of inverse rendering by enforcing geometric consistency, spatial smoothness, and structural fidelity. It differs from its sibling [Composite Image Loss with Visibility Weighting](4262-composite-image-loss-with-visibility-weighting.md) by explicitly incorporating diverse regularization terms (opacity, distortion, normal, size, flow) rather than focusing primarily on visibility-aware weighting of the photometric term.

## How it works
The method minimizes a weighted sum of a photometric loss and a set of regularization losses. The specific components vary by application (static vs. dynamic scenes, implicit vs. explicit primitives), but the general structure is:
$$ \mathcal{L} = \mathcal{L}_{\text{photo}} + \sum_i w_i \mathcal{L}_{\text{reg}, i} $$

### Inputs and Outputs
**Inputs:**
- Rendered color $\mathbf{C}_{\text{rendered}}$ (or $\mathbf{C}_{\text{Inp}}, \mathbf{C}_{\text{Up}}$).
- Ground-truth color $\mathbf{C}_{\text{gt}}$.
- Scene parameters to be optimized (e.g., voxel grid densities $\sigma$, opacities, Gaussian scales $\mathbf{s}$, motion fields $\mathbf{V}_{\text{flow}}$).
- Ray set $\mathcal{R}$ sampled for the current batch.

**Outputs:**
- Scalar loss value $\mathcal{L}$ used for gradient backpropagation.

### Data Structures
The implementation depends on the underlying representation:
1.  **Voxel/Grid-based (e.g., [S1]):**
    -   $\mathbf{V}_{\sigma}^{\text{Can}}$: 3D grid of densities in canonical space.
    -   $\mathbf{V}_{\text{flow}}^t$: 3D grid of motion vectors at time $t$.
    -   $\mathbf{D}$: Rendered depth maps.
2.  **Primitive-based (e.g., [S2], [S3, unofficial]):**
    -   $\{v_0, v_1, v_2\}$: Vertices of splatting primitives (e.g., triangles or Gaussians).
    -   $\mathbf{s}_i$: Scaling vector for primitive $i$.
    -   Opacity $\alpha_i$.

### Formulas and Steps

#### 1. Photometric Loss ($\mathcal{L}_{\text{photo}}$)
Measures the discrepancy between rendered and ground-truth images.

**Variant A: L1 + LD-SSIM ([S2])**
Used in Triangle Splatting.
$$ \mathcal{L}_{\text{photo}} = (1 - \lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{\text{D-SSIM}} $$
-   $\mathcal{L}_1$: L1 loss between rendered and ground-truth colors.
-   $\mathcal{L}_{\text{D-SSIM}}$: Learned Deep-SSIM loss [S2].
-   $\lambda$: Weight for the SSIM term.

**Variant B: Multi-Grid MSE ([S1])**
Used in dynamic voxel rendering with inpainting.
$$ \mathcal{L}_{\text{photo}} = \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \left\| \mathbf{C}_{\text{Inp}}(r) - \mathbf{C}_{\text{gt}}(r) \right\|_2^2 + \frac{1}{|\mathcal{R}|} \sum_{r \in \mathcal{R}} \left\| \mathbf{C}_{\text{Up}}(r) - \mathbf{C}_{\text{gt}}(r) \right\|_2^2 $$
-   $\mathbf{C}_{\text{Inp}}$: Rendered color from inpainted grid.
-   $\mathbf{C}_{\text{Up}}$: Rendered color from up-sampled grid without inpainting.
-   $\mathcal{R}$: Set of rays in the batch.

**Variant C: L1 + Perceptual ([S3, unofficial])**
Used in SplattingAvatar.
$$ \mathcal{L}_{\text{photo}} = \mathcal{L}_1 + \lambda_l \mathcal{L}_{\text{LPIPS}} $$
-   $\mathcal{L}_{\text{LPIPS}}$: Learned Perceptual Image Patch Similarity loss.

#### 2. Regularization Terms

**Opacity Loss ($\mathcal{L}_0$) ([S2])**
Encourages binary opacity (foreground/background separation).
$$ \mathcal{L}_0 = \sum_i \alpha_i (1 - \alpha_i) $$
-   $\alpha_i$: Opacity of primitive $i$.

**Distortion Loss ($\mathcal{L}_d$) ([S2])**
Enforces correct depth ordering along rays.

**Normal Consistency Loss ($\mathcal{L}_n$) ([S2])**
Enforces smoothness of surface normals derived from the density field.

**Size Regularization ($\mathcal{L}_s$) ([S2])**
Encourages larger triangles in triangle splatting by penalizing small cross-products.
$$ \mathcal{L}_s = 2 \| (v_1 - v_0) \times (v_2 - v_0) \|^{-1} $$
-   $v_0, v_1, v_2$: Vertices of the triangle primitive.

**Scaling Regularization ($\mathcal{L}_{\text{scaling}}$) ([S3, unofficial])**
Prevents Gaussians from becoming long and thin.
$$ \mathcal{L}_{\text{scaling}}(i) = \begin{cases} |\hat{s}_i|, & \hat{s}_i > \max(T_s, T_r \tilde{s}_i) \\ 0, & \text{otherwise} \end{cases} $$
-   $\hat{s}_i$: Maximum scaling value of Gaussian $i$.
-   $\tilde{s}_i$: Minimum scaling value of Gaussian $i$.
-   $T_s$: Threshold for length.
-   $T_r$: Threshold for thinness ratio.

**Point Color Supervision ($\mathcal{L}_{\text{ptc}}$) ([S1])**
Directly supervises the color of sampled points, weighted by their contribution to the ray.

**Background Entropy Loss ($\mathcal{L}_{\text{bg}}$) ([S1])**
Encourages densities to concentrate on either foreground or background (binary opacity).

**Flow Loss ($\mathcal{L}_{\text{flow}}$) ([S1])**
Encourages static scenes in canonical space.
$$ \mathcal{L}_{\text{flow}} = \| \mathbf{V}_{\text{flow}}^t \|_1 $$
-   $\mathbf{V}_{\text{flow}}^t$: Motion field in canonical space at time $t$.

**Inpaint Network Loss ($\mathcal{L}_{\text{vdiff}}$) ([S1])**
Ensures inpainted grid is close to input grid.
$$ \mathcal{L}_{\text{vdiff}} = \| \mathbf{V}_{\text{R}_{\text{Inp}}}^t - \mathbf{V}_{\text{R}_{\text{Up}}}^t \|_1 $$

**Total Variation Loss ($\mathcal{L}_{\text{tv}}$) ([S1])**
Enforces spatial smoothness on density, flow, or depth.
$$ \mathcal{L}_{\text{tv}}(\mathbf{X}) = \sum_{i,j} \| \mathbf{X}_{i+1,j} - \mathbf{X}_{i,j} \|_1 + \| \mathbf{X}_{i,j+1} - \mathbf{X}_{i,j} \|_1 $$
Applied to $\mathbf{V}_{\sigma}^{\text{Can}}$, $\mathbf{V}_{\text{flow}}^t$, and rendered depths $\mathbf{D}$.

#### 3. Total Loss Assembly

**Assembly A: Dynamic Voxel Rendering ([S1])**
$$ \mathcal{L} = \mathcal{L}_{\text{photo}} + w_1\mathcal{L}_{\text{ptc}} + w_2\mathcal{L}_{\text{bg}} + w_3\mathcal{L}_{\text{flow}} + w_4\mathcal{L}_{\text{vdiff}} + w_5\mathcal{L}_{\text{tv}}(\mathbf{V}_{\sigma}^{\text{Can}}) + w_6\mathcal{L}_{\text{tv}}(\mathbf{V}_{\text{flow}}^t) + w_7\mathcal{L}_{\text{tv}}(\mathbf{D}) $$
-   $w_1 \sim w_7$: Hyperparameters balancing components.

**Assembly B: Triangle Splatting ([S2])**
$$ \mathcal{L} = (1 - \lambda)\mathcal{L}_1 + \lambda\mathcal{L}_{\text{D-SSIM}} + \beta_1\mathcal{L}_0 + \beta_2\mathcal{L}_d + \beta_3\mathcal{L}_n + \beta_4\mathcal{L}_s $$
-   $\beta_1 \sim \beta_4$: Weights for opacity, distortion, normal, and size regularization.

**Assembly C: SplattingAvatar ([S3, unofficial])**
$$ \mathcal{L} = \mathcal{L}_1 + \lambda_l \mathcal{L}_{\text{LPIPS}} + \lambda_s \mathcal{L}_{\text{scaling}} $$

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_composite_loss(rendered_color, gt_color, scene_params, config):
    """
    Computes the total composite loss for volume rendering optimization.
    
    Args:
        rendered_color: Tensor of shape (H, W, 3) or (Batch, H, W, 3).
        gt_color: Tensor of shape (H, W, 3) or (Batch, H, W, 3).
        scene_params: Dict containing optimization variables (e.g., densities, scales, opacities).
        config: Dict containing loss weights and hyperparameters.
        
    Returns:
        Scalar tensor representing the total loss.
    """
    total_loss = 0.0
    
    # 1. Photometric Loss
    # Option A: L1 + SSIM (Triangle Splatting)
    if config.loss_type == "l1_dssim":
        l1 = torch.mean(torch.abs(rendered_color - gt_color))
        dssim = compute_ldssim(rendered_color, gt_color) # Implementation of LD-SSIM [S2]
        total_loss += (1 - config.lambda_) * l1 + config.lambda_ * dssim
        
    # Option B: Multi-Grid MSE (Dynamic Voxel)
    elif config.loss_type == "multi_grid_mse":
        c_inp = rendered_color['inpainted']
        c_up = rendered_color['upsampled']
        mse_inp = torch.mean((c_inp - gt_color) ** 2)
        mse_up = torch.mean((c_up - gt_color) ** 2)
        total_loss += mse_inp + mse_up
        
    # Option C: L1 + Perceptual (SplattingAvatar)
    elif config.loss_type == "l1_lpips":
        l1 = torch.mean(torch.abs(rendered_color - gt_color))
        lpips = compute_lpips(rendered_color, gt_color) # Implementation of LPIPS [S3, unofficial]
        total_loss += l1 + config.lambda_l * lpips
        
    # 2. Regularization Terms (Add based on config)
    
    # Opacity Loss (Triangle Splatting)
    if config.use_opacity_loss:
        opacities = scene_params['opacities']
        l0 = torch.mean(opacities * (1 - opacities))
        total_loss += config.beta_1 * l0
        
    # Distortion Loss (Triangle Splatting)
    if config.use_distortion_loss:
        l_d = compute_distortion_loss(scene_params['depths'], scene_params['weights']) # [S2]
        total_loss += config.beta_2 * l_d
        
    # Normal Loss (Triangle Splatting)
    if config.use_normal_loss:
        l_n = compute_normal_consistency_loss(scene_params['normals']) # [S2]
        total_loss += config.beta_3 * l_n
        
    # Size Regularization (Triangle Splatting)
    if config.use_size_reg:
        vertices = scene_params['vertices'] # (N, 3, 3) for N triangles
        v0, v1, v2 = vertices[:, 0], vertices[:, 1], vertices[:, 2]
        cross_prod = torch.cross(v1 - v0, v2 - v0)
        l_s = torch.mean(2.0 / (torch.norm(cross_prod, dim=1) + 1e-8))
        total_loss += config.beta_4 * l_s
        
    # Scaling Regularization (SplattingAvatar)
    if config.use_scaling_reg:
        scales = scene_params['scales'] # (N, 3)
        max_s = torch.max(scales, dim=1)[0]
        min_s = torch.min(scales, dim=1)[0]
        threshold = torch.max(config.T_s, config.T_r * min_s)
        mask = max_s > threshold
        l_scaling = torch.mean(torch.abs(max_s[mask]))
        total_loss += config.lambda_s * l_scaling
        
    # Point Color Supervision (Dynamic Voxel)
    if config.use_ptc:
        l_ptc = compute_point_color_loss(scene_params['sampled_colors'], scene_params['weights']) # [S1]
        total_loss += config.w_1 * l_ptc
        
    # Background Entropy Loss (Dynamic Voxel)
    if config.use_bg_loss:
        densities = scene_params['densities']
        l_bg = compute_entropy_loss(densities) # [S1]
        total_loss += config.w_2 * l_bg
        
    # Flow Loss (Dynamic Voxel)
    if config.use_flow_loss:
        flow = scene_params['flow_field']
        l_flow = torch.mean(torch.abs(flow))
        total_loss += config.w_3 * l_flow
        
    # Inpaint Network Loss (Dynamic Voxel)
    if config.use_vdiff:
        v_inp = scene_params['inpainted_grid']
        v_up = scene_params['upsampled_grid']
        l_vdiff = torch.mean(torch.abs(v_inp - v_up))
        total_loss += config.w_4 * l_vdiff
        
    # Total Variation Loss (Dynamic Voxel)
    if config.use_tv_loss:
        # TV on Density
        l_tv_sigma = compute_total_variation(scene_params['density_grid'])
        total_loss += config.w_5 * l_tv_sigma
        
        # TV on Flow
        l_tv_flow = compute_total_variation(scene_params['flow_grid'])
        total_loss += config.w_6 * l_tv_flow
        
        # TV on Depth
        l_tv_depth = compute_total_variation(scene_params['depth_map'])
        total_loss += config.w_7 * l_tv_depth
        
    return total_loss
```

### Parameters
-   **$\lambda$**: Weight for LD-SSIM term. Typical values not specified in sources, but must be in $[0, 1]$.
-   **$\beta_1, \beta_2, \beta_3, \beta_4$**: Weights for opacity, distortion, normal, and size regularization respectively. Values detailed in supplementary materials of [S2].
-   **$w_1 \sim w_7$**: Weights for dynamic voxel regularization components. Values not explicitly given, but used to balance components [S1].
-   **$T_s, T_r$**: Thresholds for scaling regularization length and thinness ratio. Values not explicitly given [S3, unofficial].
-   **$\lambda_l, \lambda_s$**: Weights for LPIPS and Scaling regularization in SplattingAvatar. Values not explicitly given [S3, unofficial].

### Edge Cases and Pitfalls
-   **Precision**: Inverse operations in size regularization ($\| \dots \|^{-1}$) require epsilon guards to avoid division by zero for degenerate triangles [S2].
-   **Boundaries**: Total variation loss requires handling grid boundaries correctly (e.g., zero-padding or periodic boundary conditions) [S1].
-   **Degenerate Input**: If rendered and ground-truth colors are identical, photometric loss is zero, but regularization terms still provide gradients. If primitives are degenerate (zero area/volume), size/flow losses may become unstable.

## Subtopics

### Scaling and Background Mixing Regularization
A variant used in monocular dynamic Gaussian splatting (SplattingAvatar) to address limited viewing angles. It introduces a random background color mixed with the rendered and ground-truth images to provide silhouette cues. It adds a scaling regularization term $\mathcal{L}_{\text{scaling}}$ that penalizes Gaussians that are both long (max scaling $> T_s$) and thin (max scaling $> T_r \times$ min scaling) [S3, unofficial]. This prevents the Gaussians from growing excessively in one dimension to fit the monocular data.

## Operating characteristics
-   **Time Complexity**: Dominated by the rendering pass and the computation of photometric loss ($O(N)$ where $N$ is the number of pixels/rays). Regularization terms add overhead proportional to the number of primitives or grid voxels.
-   **Memory Complexity**: Requires storing gradients for all optimization variables (densities, scales, opacities, etc.). Total variation loss requires storing intermediate gradient maps.
-   **Strengths**: Provides robust optimization by combining multiple priors, preventing common artifacts like floating artifacts (opacity loss), jittering (flow loss), or unnatural shapes (size/normal loss).
-   **Limitations**: Requires careful tuning of multiple hyperparameters ($w_i, \beta_i, \lambda$). Computationally expensive due to multiple loss terms.
-   **When to use**: When optimizing complex radiance fields or splatting models from limited or noisy data (e.g., monocular video, sparse views).
-   **When not to use**: For simple, well-lit, multi-view static scenes where basic L1/L2 photometric loss may suffice.

## Background
The concept of combining photometric and regularization losses is fundamental to differentiable rendering. The specific formulations cited here originate from recent advances in radiance field optimization:
-   [S1] (2023) introduced a comprehensive set of regularization terms for dynamic voxel-based rendering, including flow and inpainting losses.
-   [S2] (2026) proposed size and normal regularization for triangle-based splatting.
-   [S3, unofficial] (2024) introduced scaling regularization and background mixing for avatar reconstruction from monocular video.

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
[S2] Triangle Splatting for Real-Time Radiance Field Rendering, https://doi.org/10.1109/3DV69130.2026.00123, fetched 2026-10-27
[S3] SplattingAvatar, https://arxiv.org/abs/2403.05087, fetched 2026-10-27 (unofficial)
