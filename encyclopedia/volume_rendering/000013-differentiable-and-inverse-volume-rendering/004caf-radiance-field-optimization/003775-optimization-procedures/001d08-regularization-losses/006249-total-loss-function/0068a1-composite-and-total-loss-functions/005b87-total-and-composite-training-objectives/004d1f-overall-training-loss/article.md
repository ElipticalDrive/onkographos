# Overall Training Loss

Also called: End-to-End Training Objective, Multi-term Loss Function, PDRF Training Loss, Total Training Loss

## Overview
The overall training loss is the composite objective function used to optimize differentiable volume rendering representations (such as Gaussian Splatting, Voxel Grids, or Implicit Neural Fields) by aggregating multiple supervision signals into a single scalar value [10621-overall-training-loss.md]. It solves the problem of balancing photometric fidelity, geometric consistency, semantic alignment, and regularization constraints during end-to-end training [S3][S6, unofficial]. What sets this method apart from its siblings is its role as the final aggregation layer in the optimization procedure, where specific loss components (rendering, depth, semantic, regularization) are weighted and summed according to the chosen representation and training stage [10621-overall-training-loss.md].

## How it works
The method constructs the total loss $L_{tot}$ via the weighted summation of individual loss components $\mathcal{L}_k$, as defined in the parent [Total and Composite Training Objectives](16924-total-and-composite-training-objectives.md):

$$ L_{tot} = \sum_{k} \lambda_k \mathcal{L}_k $$

The specific instantiation of this formula varies based on the representation, training stage, and scale [10621-overall-training-loss.md]. Below are the detailed implementations found in the sources.

### Inputs and Outputs
*   **Inputs:** Rendered images/volumes ($\hat{I}, \hat{V}$), Ground truth images/volumes ($I, V$), Camera poses ($P$), Depth maps ($D$), Semantic features/labels, and Mask ($M$).
*   **Outputs:** Scalar loss value $L_{tot}$ used for gradient backpropagation.

### Data Structures
*   **Loss Components:** Individual loss terms ($\mathcal{L}_k$) are computed per-pixel or per-voxel and then aggregated (summed or averaged).
*   **Weights:** Scalar hyperparameters $\lambda_k$ control the contribution of each term.

### Formulas and Steps
The following formulas represent distinct instantiations of the overall training loss found in the literature.

**1. PDRF (Progressively Deblurring Radiance Field) [S1]**
Used for deblurring scene reconstruction from blurry images.
$$ L_{tot} = L_{CRR\_photo} + L_{FVR\_photo} + \lambda (L_{TV\_coar} + L_{TV\_fine}) $$
*   $L_{CRR\_photo}$: Photometric loss for Coarse Radiance Field (CRR).
*   $L_{FVR\_photo}$: Photometric loss for Fine Voxel Renderer (FVR).
*   $L_{TV\_coar}, L_{TV\_fine}$: Total variation constraints on explicit feature volumes to enforce smoothness.
*   $\lambda$: Weight for total variation.
*   **Parameters:** Batch size: 1024 rays. Gamma correction: 2.2.

**2. TD-NeRF (Truncated Depth Prior) [S2]**
Used for joint camera pose and neural radiance field optimization.
$$ L = L_{rgb} + \lambda_1 L_{self-depth} + \lambda_2 L_{GPC} + \lambda_3 L_{reproj} $$
*   $L_{rgb}$: RGB photometric loss.
*   $L_{self-depth}$: Self-supervised depth loss.
*   $L_{GPC}$: Gaussian Point Constraint loss (based on Gaussian kernel function).
*   $L_{reproj}$: Reprojection loss.
*   **Parameters:** $\lambda_1 = 0.04, \lambda_2 = 1.0, \lambda_3 = 1.0$.

**3. Compact 3D Gaussian Representation [S3]**
Used for end-to-end training of Gaussian splatting with codebooks.
$$ L = L_{ren} + \lambda_m L_m + L_r + L_s $$
*   $L_{ren}$: Rendering loss (weighted sum of L1 and SSIM loss between GT and rendered images).
*   $L_m$: Masking loss.
*   $L_r$: Rotation codebook loss.
*   $L_s$: Scale codebook loss.
*   $\lambda_m$: Hyperparameter to regularize the number of Gaussians.
*   **Note:** $L_r$ and $L_s$ are applied only for the last 1K training iterations to avoid heavy computation [S3].

**4. ZipSplat [S6, unofficial]**
Used for refining cameras and scene tokens with rendering, geometric, and camera supervision.
$$ \mathcal{L} = \mathcal{L}_{\mathrm{render}} + \lambda_{\mathrm{geo}}\,\mathcal{L}_{\mathrm{geo}} + \lambda_{\mathrm{cone}}\,\mathcal{L}_{\mathrm{cone}} + \lambda_{\mathrm{pose}}\,\mathcal{L}_{\mathrm{pose}} $$
*   $\mathcal{L}_{\mathrm{render}}$: Combines $\ell_1$ and perceptual (LPIPS) terms.
*   $\mathcal{L}_{\mathrm{geo}}$: Geometric loss.
*   $\mathcal{L}_{\mathrm{cone}}$: Cone loss.
*   $\mathcal{L}_{\mathrm{pose}}$: Pose supervision loss.
*   **Parameters:** LPIPS weight: 0.1. $\lambda_{geo} = 0.05, \lambda_{cone} = 0.01, \lambda_{pose} = 0.05$.

**5. ReSplat (Recurrent Gaussian Splatting) [S4]**
Uses a two-stage training process.
*   **Stage 1 (Initialization):**
    $$ L_{1st} = \sum_{v=1}^V \ell_{\text{render}}(\hat{I}_v, I_v) + \alpha \cdot \sum_{i=1}^N \ell_{\text{smooth}}(I_i, \hat{D}_i) $$
    *   $\ell_{\text{render}}(\hat{I}, I) = \ell_1(\hat{I}, I) + \lambda \cdot \ell_{\text{percept}}(\hat{I}, I)$.
    *   $\ell_{\text{smooth}}(I, \hat{D}) = |\partial_x \hat{D}| e^{-|\partial_x I|} + |\partial_y \hat{D}| e^{-|\partial_y I|}$.
    *   **Parameters:** $\alpha = 0.01, \lambda = 0.5$.
*   **Stage 2 (Recurrent Training):**
    $$ L_{2nd} = \sum_{t=0}^{T-1} \gamma^{T-1-t} \sum_{v=1}^V \ell_{\text{render}}(\hat{I}_v^t, I_v) $$
    *   Supervises all Gaussian predictions during the recurrent process with exponentially increasing weights.
    *   **Parameters:** $\gamma = 0.9$.

**6. OV-NeRF (Open-Vocabulary NeRF) [S5]**
Uses a two-stage semantic field training process.
*   **Stage I (Initial Training):**
    $$ L_{Total} = L_{color} + L_{feat} + L_{dino} + L_{RSR} $$
    *   $L_{RSR}$: Region semantic ranking regularization loss.
*   **Stage II (Cross-View Enhancement):**
    $$ L_{Total} = L_{color} + L_{feat} + L_{dino} + L_{CSE} $$
    *   $L_{CSE}$: Cross-view self-enhancement loss.
    *   **Parameters:** $T = 10,000$ iterations before switching to Stage II.

**7. X-GRM (X-ray Gaussian Reconstruction Model) [S8, unofficial]**
Uses volume and rendering constraints.
$$ \mathcal{L} = \mathcal{L}_{\text{volume}} + \mathcal{L}_{\text{render}} $$
*   $\mathcal{L}_{\text{volume}}(\mathbf{V}, \mathbf{V}_{gt}) = \sum_{x,y,z} (\mathbf{V}(x, y, z) - \mathbf{V}_{gt}(x, y, z))^2$.
*   $\mathcal{L}_{\text{render}}(\mathbf{I}, \mathbf{I}_{gt}) = \lambda_{L1}\mathcal{L}_1(\mathbf{I}, \mathbf{I}_{gt}) + \lambda_{ssim}\mathcal{L}_{ssim}(\mathbf{I}, \mathbf{I}_{gt})$.
*   **Parameters:** Volume resolution: $256^3$. Sub-volume sampling factor: 1/4 in each dimension.

**8. VoxelTTO [S9, unofficial]**
Combines rendered-depth and rendered-RGB losses.
$$ L = L_{depth} + L_{rgb} = \lambda_{\ell1}^d L_{\ell1}^{depth} + \lambda_{align} L_{align}^{depth} + \lambda_{\ell1} L_{\ell1}^{rgb} + \lambda_{ssim} L_{ssim}^{rgb} $$
*   $L_{\ell1}^{depth}$: Supervises rendered depth with ground-truth depth.
*   $L_{align}^{depth}$: Enforces alignment with VFM-predicted depth.
*   $L_{\ell1}^{rgb}, L_{ssim}^{rgb}$: $\ell_1$ and SSIM losses between rendered and ground-truth images.
*   **Parameters:** $\lambda_{\ell1}^d = 0.25, \lambda_{align} = 0.25, \lambda_{\ell1} = 0.9, \lambda_{ssim} = 1.0$.

### Pseudocode
The following pseudocode illustrates the general structure of computing an overall training loss, incorporating elements from the sources. It assumes a generic differentiable renderer and specific loss components.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_overall_loss(rendered_image, gt_image, rendered_depth, gt_depth, 
                         mask, semantic_features, gt_semantic, camera_poses, 
                         gt_camera_poses, volume=None, gt_volume=None, 
                         stage='default', iteration=0):
    """
    Computes the total training loss based on the selected method variant.
    
    Args:
        rendered_image: Rendered RGB image.
        gt_image: Ground truth RGB image.
        rendered_depth: Rendered depth map.
        gt_depth: Ground truth depth map (if available).
        mask: Binary mask for valid pixels (1 for valid, 0 for background).
        semantic_features: Semantic features from the network.
        gt_semantic: Ground truth semantic labels (if available).
        camera_poses: Predicted camera poses.
        gt_camera_poses: Ground truth camera poses (if available).
        volume: Predicted volume (for voxel-based methods).
        gt_volume: Ground truth volume (for voxel-based methods).
        stage: Training stage ('init', 'recurrent', 'semantic_i', 'semantic_ii').
        iteration: Current training iteration.
        
    Returns:
        total_loss: Scalar loss value.
    """
    total_loss = 0.0
    
    # 1. Photometric Rendering Loss (Common across most methods)
    # L1 Loss
    l1_loss = torch.mean(torch.abs(rendered_image - gt_image))
    
    # SSIM Loss (Structural Similarity)
    ssim_loss = 1.0 - ssim(rendered_image, gt_image)
    
    # Perceptual Loss (LPIPS) - Optional, used in ZipSplat, ReSplat
    # perceptual_loss = lpips(rendered_image, gt_image)
    
    # Combine photometric terms
    lambda_l1 = 0.9
    lambda_ssim = 1.0
    lambda_percept = 0.1 # Used in ZipSplat, ReSplat
    
    rendering_loss = lambda_l1 * l1_loss + lambda_ssim * ssim_loss
    # rendering_loss += lambda_percept * perceptual_loss # If perceptual is used
    
    # Apply Loss Masking if mask is provided [S7]
    if mask is not None:
        rendering_loss = rendering_loss * mask
        
    total_loss += rendering_loss
    
    # 2. Depth Losses
    if gt_depth is not None:
        # L1 Depth Loss
        l1_depth_loss = torch.mean(torch.abs(rendered_depth - gt_depth))
        lambda_l1_depth = 0.25 # VoxelTTO
        
        # Alignment Loss (with VFM predictions) - VoxelTTO
        # vfm_depth = get_vfm_depth(rendered_image) # Placeholder for VFM prediction
        # align_depth_loss = torch.mean(torch.abs(rendered_depth - vfm_depth))
        # lambda_align = 0.25
        # depth_loss = lambda_l1_depth * l1_depth_loss + lambda_align * align_depth_loss
        depth_loss = lambda_l1_depth * l1_depth_loss
        
        total_loss += depth_loss
        
        # Self-Supervised Depth Loss (TD-NeRF) [S2]
        # lambda_self_depth = 0.04
        # self_depth_loss = compute_self_supervised_depth_loss(rendered_depth, rendered_image)
        # total_loss += lambda_self_depth * self_depth_loss
        
        # Depth Smoothness Loss (ReSplat Stage 1) [S4]
        # alpha = 0.01
        # smooth_loss = compute_depth_smoothness(rendered_image, rendered_depth)
        # if stage == 'init':
        #     total_loss += alpha * smooth_loss

    # 3. Geometric and Pose Losses
    if gt_camera_poses is not None:
        # Pose Loss (ZipSplat) [S6, unofficial]
        # lambda_pose = 0.05
        # pose_loss = compute_pose_error(camera_poses, gt_camera_poses)
        # total_loss += lambda_pose * pose_loss
        
        # Reprojection Loss (TD-NeRF) [S2]
        # lambda_reproj = 1.0
        # reproj_loss = compute_reprojection_loss(camera_poses, rendered_image, gt_image)
        # total_loss += lambda_reproj * reproj_loss
        
        # Gaussian Point Constraint (TD-NeRF) [S2]
        # lambda_gpc = 1.0
        # gpc_loss = compute_gaussian_point_constraint(rendered_depth, camera_poses)
        # total_loss += lambda_gpc * gpc_loss

    # 4. Semantic Losses (OV-NeRF) [S5]
    if gt_semantic is not None:
        # Color/Feature Loss (Semantic consistency)
        # l_color = compute_semantic_color_loss(semantic_features, gt_semantic)
        # l_feat = compute_feature_loss(semantic_features, gt_semantic)
        # l_dino = compute_dino_loss(semantic_features)
        
        if stage == 'semantic_i':
            # L_RSR: Region Semantic Ranking Regularization
            # l_rsr = compute_rsr_loss(semantic_features)
            # total_loss += l_rsr
            pass
        elif stage == 'semantic_ii':
            # L_CSE: Cross-View Self-Enhancement
            # l_cse = compute_cse_loss(semantic_features)
            # total_loss += l_cse
            pass

    # 5. Volume Losses (X-GRM) [S8, unofficial]
    if volume is not None and gt_volume is not None:
        # MSE Volume Loss
        volume_loss = torch.mean((volume - gt_volume) ** 2)
        total_loss += volume_loss

    # 6. Regularization Losses
    # Total Variation (PDRF) [S1]
    # lambda_tv = 1.0
    # tv_coar = compute_total_variation(coar_volume)
    # tv_fine = compute_total_variation(fine_volume)
    # total_loss += lambda_tv * (tv_coar + tv_fine)
    
    # Codebook Losses (Compact 3D Gaussian) [S3]
    # if iteration > total_iterations - 1000:
    #     lambda_m = 0.1 # Example value
    #     l_m = compute_masking_loss(mask)
    #     l_r = compute_rotation_codebook_loss(rotations)
    #     l_s = compute_scale_codebook_loss(scales)
    #     total_loss += lambda_m * l_m + l_r + l_s

    return total_loss

def compute_depth_smoothness(image, depth):
    """
    Computes depth smoothness loss based on image gradients.
    L_smooth = |d_x D| * exp(-|d_x I|) + |d_y D| * exp(-|d_y I|)
    """
    # Compute gradients
    depth_x = torch.abs(depth[:, :, 1:] - depth[:, :, :-1])
    depth_y = torch.abs(depth[:, 1:, :] - depth[:, :-1, :])
    
    image_x = torch.abs(image[:, :, 1:] - image[:, :, :-1])
    image_y = torch.abs(image[:, 1:, :] - image[:, :-1, :])
    
    # Ensure dimensions match
    depth_x = torch.nn.functional.pad(depth_x, (0, 0, 0, 1))
    depth_y = torch.nn.functional.pad(depth_y, (0, 1, 0, 0))
    
    smooth_loss = depth_x * torch.exp(-image_x) + depth_y * torch.exp(-image_y)
    return torch.mean(smooth_loss)
```

### Parameters with Typical Values
*   **L1/SSIM Weights:** $\lambda_{L1} \approx 0.9, \lambda_{SSIM} \approx 1.0$ [S9, unofficial].
*   **Perceptual (LPIPS) Weight:** 0.1 [S6, unofficial].
*   **Depth Weights:** $\lambda_{depth} \approx 0.25$ [S9, unofficial].
*   **Self-Supervised Depth Weight:** 0.04 [S2].
*   **GPC Weight:** 1.0 [S2].
*   **Reprojection Weight:** 1.0 [S2].
*   **Pose Weight:** 0.05 [S6, unofficial].
*   **Geometric Weight:** 0.05 [S6, unofficial].
*   **Cone Weight:** 0.01 [S6, unofficial].
*   **Depth Smoothness Weight ($\alpha$):** 0.01 [S4].
*   **Perceptual Loss Weight ($\lambda$):** 0.5 [S4].
*   **Masking Regularization ($\lambda_m$):** Not given in the sources.
*   **Total Variation Weight ($\lambda$):** Not given in the sources.
*   **Recurrent Weight Decay ($\gamma$):** 0.9 [S4].
*   **Semantic Stage Iterations ($T$):** 10,000 [S5].

### Edge Cases and Pitfalls
*   **Masking:** Without loss masking, the network must match default ground truth values in empty regions (background), which complicates the task [S7]. Loss masking simplifies this by ignoring these areas.
*   **Codebook Training:** Applying codebook losses ($L_r, L_s$) for the entire training duration causes heavy computation. It is recommended to apply them only for the last 1K iterations [S3].
*   **Memory Usage:** Rendering with full volume resolution ($256^3$) requires exhaustive GPU memory to preserve gradients. Randomly sampling sub-volumes (e.g., $1/4$ in each dimension) reduces memory usage during training [S8, unofficial].
*   **Depth Sensitivity:** Chamfer distance for point cloud constraints can be sensitive to noise in estimated depth maps. Gaussian Point Constraint (GPC) is proposed as a more robust alternative [S2].

## Subtopics

### Loss Masking
A technique to modulate loss terms for normals, ambient occlusion, and colors with the output mask, so that masked-out pixels (background) do not contribute to the loss function [S7]. It simplifies the network's task by ignoring areas where the ground truth images are filled with default values [S7].

### X-GRM Training Objectives
A training objective for X-ray Gaussian Reconstruction Models that uses volume constraints (MSE) and rendering constraints (L1 + SSIM) [S8, unofficial]. It employs random sub-volume sampling to reduce GPU memory usage during training [S8, unofficial].

### VoxelTTO Training Loss
A training loss for Voxel-Aligned Feed-Forward 3D Gaussian Splatting that combines rendered-depth loss and rendered-RGB loss [S9, unofficial]. It supervises depth alignment with VFM predictions and RGB appearance with ground truth [S9, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of rays/samples per iteration and the complexity of the loss terms (e.g., LPIPS requires feature extraction). PDRF uses importance sampling to reduce samples [S1].
*   **Memory Complexity:** High for voxel-based methods due to gradient storage. Sub-volume sampling can mitigate this [S8, unofficial]. Gaussian Splatting methods generally have lower memory overhead than dense voxel grids.
*   **Strengths:** Flexible composition of multiple supervision signals; allows for end-to-end optimization of complex representations; handles various tasks (rendering, depth, semantics) jointly.
*   **Limitations:** Requires careful tuning of hyperparameters ($\lambda_k$); some loss terms (e.g., perceptual, codebook) are computationally expensive; sensitive to initialization and training stages.
*   **When to Use:** When optimizing differentiable volume rendering representations with multiple supervision signals (photometric, geometric, semantic).
*   **When Not to Use:** When computational resources are severely limited and simple photometric loss is sufficient; when ground truth for specific loss terms (e.g., depth, semantics) is unavailable.

## Background
The concept of combining multiple loss terms for training differentiable rendering models has evolved alongside the development of Neural Radiance Fields (NeRF) and 3D Gaussian Splatting. Early NeRF methods primarily used photometric loss [S1]. As the field advanced, additional constraints such as depth, pose, and semantic information were incorporated to improve robustness and generalization [S2, S5][S6, unofficial]. The specific formulations of the overall training loss vary significantly depending on the representation (Gaussian, Voxel, Implicit) and the specific task (deblurring, sparse-view reconstruction, open-vocabulary understanding) [10621-overall-training-loss.md].

## Sources
[S1] PDRF: Progressively Deblurring Radiance Field for Fast Scene Reconstruction from Blurry Images, https://doi.org/10.1609/aaai.v37i2.25295, fetched 2026-10-26
[S2] TD-NeRF: Novel Truncated Depth Prior for Joint Camera Pose and Neural Radiance Field Optimization, https://doi.org/10.1109/iros58592.2024.10802634, fetched 2026-10-26
[S3] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S4] ReSplat: Learning Recurrent Gaussian Splatting, https://arxiv.org/abs/2510.08575v4, fetched 2026-10-26
[S5] OV-NeRF: Open-Vocabulary Neural Radiance Fields With Vision and Language Foundation Models for 3D Semantic Understanding, https://doi.org/10.1109/tcsvt.2024.3439737, fetched 2026-10-26
[S6] ZipSplat: Fewer Gaussians, Better Splats, https://doi.org/10.48550/arXiv.2606.05102, fetched 2026-10-29 (unofficial)
[S7] Volumetric Isosurface Rendering with Deep Learning-Based Super ..., https://ieeexplore.ieee.org/abstract/document/8918030, fetched 2026-10-31
[S8] X-GRM: Large Gaussian Reconstruction Model for Sparse-view X-rays to Computed Tomography, https://doi.org/10.48550/arXiv.2505.15235, fetched 2026-10-27 (unofficial)
[S9] VoxelTTO: Voxel-Aligned Feed-Forward 3D Gaussian Splatting with Test-Time Optimization, https://doi.org/10.48550/arxiv.2609.21498, fetched 2026-10-26 (unofficial)
