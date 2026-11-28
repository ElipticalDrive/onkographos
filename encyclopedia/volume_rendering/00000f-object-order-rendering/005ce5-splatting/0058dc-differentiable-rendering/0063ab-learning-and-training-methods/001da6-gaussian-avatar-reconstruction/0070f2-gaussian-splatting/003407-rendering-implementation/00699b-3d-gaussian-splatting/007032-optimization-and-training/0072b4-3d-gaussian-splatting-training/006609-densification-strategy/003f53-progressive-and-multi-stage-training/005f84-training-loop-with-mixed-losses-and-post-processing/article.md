# Training Loop with Mixed Losses and Post-Processing

Also called: Mixed Loss Training Loop, Post-Processing Training Step

## Overview
This method defines the iterative optimization step for 3D Gaussian Splatting, integrating photometric reconstruction, depth supervision, appearance regularization, and structural constraints into a single backward pass. It distinguishes itself from simpler training loops by explicitly supporting multi-stage parameter updates (such as spherical harmonic degree progression and bilateral grid refinement) and post-processing densification triggers within the same iteration [S1, unofficial]. It serves as the core engine for the progressive training strategies described in [Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md).

## How it works
### Inputs and Outputs
**Inputs:**
- Current optimization `step` (integer).
- Gaussian set $\mathcal{G}$ (means, scales, quaternions, opacities, spherical harmonic coefficients).
- Camera parameters (intrinsics, extrinsics) and image IDs.
- Ground truth images $I_{gt}$ and optional depth maps $D_{gt}$.
- Configuration object `cfg` containing boolean flags and scalar weights.
- Optimization state (Adam optimizers, schedulers, bilateral grid parameters if enabled).

**Outputs:**
- Updated Gaussian parameters $\mathcal{G}'$.
- Updated camera parameters (if optimized).
- Updated bilateral grid weights (if enabled).
- Logging metrics (loss values, PSNR/SSIM/LPIPS if evaluated).

### Data Structures
The Gaussian set $\mathcal{G}$ is typically stored in packed or unpacked tensors:
- `means`: $(N, 3)$ float32.
- `scales`: $(N, 3)$ float32.
- `quats`: $(N, 4)$ float32 (normalized).
- `opacities`: $(N, 1)$ float32.
- `sh0`: $(N, 3)$ float32 (SH degree 0).
- `shN`: $(N, K)$ float32 (SH degree $>0$), where $K = 3 \times d^2 - 3$ for max degree $d$.
- `colors`: $(N, 3)$ float32 (optional, used if SH is not used for base color).

The `info` dictionary returned by the rasterizer contains:
- `radii`: $(N, 2)$ int32 (bounding box of splats in screen space).
- `gaussian_ids`: $(H, W)$ int32 (index of the top Gaussian per pixel).
- `alphas`: $(H, W, 1)$ float32 (accumulated opacity).
- `depths`: $(H, W, 1)$ float32 (accumulated depth, if requested).

### Parameters
- `cfg.sh_degree_interval`: Integer, steps between increasing SH degree.
- `cfg.sh_degree`: Integer, maximum SH degree (typically 3 or 4).
- `cfg.depth_loss`: Boolean, enable depth supervision.
- `cfg.use_bilateral_grid`: Boolean, enable appearance refinement via bilateral grids.
- `cfg.random_bkgd`: Boolean, composite against random background.
- `cfg.use_l1_for_real_samples`: Boolean, use L1 instead of L2 for reconstruction.
- `cfg.ssim_lambda`: Float, weight for SSIM loss (typically 0.2).
- `cfg.depth_lambda`: Float, weight for depth loss.
- `cfg.opacity_reg`: Float, weight for opacity regularization.
- `cfg.scale_reg`: Float, weight for scale regularization.
- `cfg.visible_adam`: Boolean, use visibility-aware Adam updates.
- `cfg.packed`: Boolean, use packed rasterization.

### Numbered Steps

1.  **Spherical Harmonic Degree Selection**:
    Calculate the active SH degree $d_{active}$ for the current step:
    $$ d_{active} = \min\left(\left\lfloor \frac{\text{step}}{\text{cfg.sh\_degree\_interval}} \right\rfloor, \text{cfg.sh\_degree}\right) $$
    This ensures gradual introduction of high-frequency appearance details [S1, unofficial].

2.  **Rasterization**:
    Render the scene using the current Gaussian set and camera parameters.
    - If `cfg.depth_loss` is true, render mode is `'RGB+ED'` (RGB + Estimated Depth).
    - Otherwise, render mode is `'RGB'`.
    - Let `renders` be the output tensor of shape $(H, W, C)$.
    - If `renders` has 4 channels, split into `colors = renders[..., 0:3]` and `depths = renders[..., 3:4]`.
    - Otherwise, `colors = renders` and `depths = None` [S1, unofficial].

3.  **Bilateral Grid Slicing (Optional)**:
    If `cfg.use_bilateral_grid` is true:
    - Compute grid coordinates `grid_xy` for each pixel based on screen position and depth.
    - Apply bilateral grid slicing to refine colors:
      $$ \text{colors} = \text{slice}(\text{self.bil\_grids}, \text{grid\_xy}, \text{colors}, \text{image\_ids})[\text{"rgb"}] $$
    - This step requires the bilateral grid optimizer to be stepped separately [S1, unofficial].

4.  **Background Compositing (Optional)**:
    If `cfg.random_bkgd` is true:
    - Generate a random background color $\mathbf{b} \sim \mathcal{U}(0, 1)$ of shape $(1, 3)$.
    - Composite the rendered colors against the background using the accumulated alpha $\alpha$:
      $$ \mathbf{c}_{final} = \mathbf{c}_{rendered} + \mathbf{b} \cdot (1.0 - \alpha) $$
    - This handles transparent regions by filling them with a random color, preventing the model from learning to render black backgrounds [S1, unofficial].

5.  **Pre-Backward Strategy Step**:
    Call `strategy.step_pre_backward(params, optimizers, state, step, info)`.
    - This allows custom strategies (e.g., MCMC or default densification logic) to modify parameters or state before loss computation [S1, unofficial].

6.  **Reconstruction Loss**:
    Compute the photometric loss between the rendered colors $\mathbf{c}_{final}$ and ground truth $I_{gt}$.
    - If the sample is not a NeRF sample (i.e., real image):
      - If `cfg.use_l1_for_real_samples` is true, use L1 loss: $L_{L1} = \|\mathbf{c}_{final} - I_{gt}\|_1$.
      - Otherwise, use L2 loss: $L_{L2} = \|\mathbf{c}_{final} - I_{gt}\|_2^2$.
    - Compute SSIM loss $L_{SSIM}$ using a differentiable SSIM implementation.
    - Combine:
      $$ L_{recon} = L_{recon\_base} \cdot (1.0 - \text{cfg.ssim\_lambda}) + L_{SSIM} \cdot \text{cfg.ssim\_lambda} $$
    - Add to total loss: $L_{total} = L_{recon}$ [S1, unofficial].

7.  **Depth Loss (Optional)**:
    If `cfg.depth_loss` is true and depth maps are available:
    - Normalize 3D points to $[-1, 1]$ range.
    - Sample depths from the ground truth depth map $D_{gt}$ using `F.grid_sample`.
    - Compute disparity from rendered depths: $d_{rendered} = 1.0 / \text{depths}$ (where $\text{depths} > 0$).
    - Compute disparity from ground truth: $d_{gt} = 1.0 / D_{gt}$.
    - Compute L1 loss on disparity, scaled by scene scale:
      $$ L_{depth} = \text{F.l1\_loss}(d_{rendered}, d_{gt}) \cdot \text{self.scene\_scale} $$
    - Add to total loss: $L_{total} += L_{depth} \cdot \text{cfg.depth\_lambda}$ [S1, unofficial].

8.  **Bilateral Grid Total Variation (Optional)**:
    If `cfg.use_bilateral_grid` is true:
    - Compute total variation loss on the bilateral grid weights:
      $$ L_{TV} = 10 \cdot \text{total\_variation\_loss}(\text{self.bil\_grids.grids}) $$
    - Add to total loss: $L_{total} += L_{TV}$ [S1, unofficial].

9.  **Opacity Regularization**:
    If `cfg.opacity_reg > 0.0`:
    - Compute mean of sigmoid-applied opacities:
      $$ L_{opacity} = \text{cfg.opacity\_reg} \cdot \mathbb{E}[\sigma(\text{opacities})] $$
    - Add to total loss: $L_{total} += L_{opacity}$ [S1, unofficial].

10. **Scale Regularization**:
    If `cfg.scale_reg > 0.0`:
    - Compute mean of exponential-applied scales:
      $$ L_{scale} = \text{cfg.scale\_reg} \cdot \mathbb{E}[\exp(\text{scales})] $$
    - Add to total loss: $L_{total} += L_{scale}$ [S1, unofficial].

11. **Backward Pass**:
    - Call `L_total.backward()`.

12. **Visibility Mask Computation (Optional)**:
    If `cfg.visible_adam` is true:
    - If `cfg.packed` is true, create a zero tensor of shape equal to the number of Gaussians. Scatter 1s at indices `info["gaussian_ids"]` for pixels with valid depth/alpha.
    - Else, compute mask as `(info["radii"] > 0).all(-1).any(0)`.
    - This mask identifies Gaussians that contributed to the final image [S1, unofficial].

13. **Optimizer Step**:
    - If `cfg.visible_adam` is true, pass the visibility mask to the optimizer:
      $$ \text{optimizer.step}(\text{visibility\_mask}) $$
    - Else:
      $$ \text{optimizer.step}() $$
    - Zero gradients with `set_to_none=True` [S1, unofficial].

14. **Pose and Bilateral Grid Optimization**:
    - Step `pose_optimizers` and `bil_grid_optimizers` if they exist.
    - Zero their gradients [S1, unofficial].

15. **Scheduler Step**:
    - Step all learning rate schedulers [S1, unofficial].

16. **Post-Backward Strategy Step**:
    - Call `strategy.step_post_backward`.
    - If `DefaultStrategy`, pass `packed=cfg.packed`.
    - If `MCMCStrategy`, pass `lr=schedulers[0].get_last_lr()[0]`.
    - This handles densification (splitting/cloning) and pruning based on gradients or other metrics [S1, unofficial].

17. **PLY Export (Optional)**:
    - If `step` is in `cfg.ply_steps` or `step == max_steps - 1` and `cfg.save_ply` is true:
      - If `cfg.app_opt` (appearance optimization) is true:
        - Bake appearance by evaluating the appearance module at the origin (directions $\mathbf{d} = \mathbf{0}$).
        - Add `self.splats["colors"]`, apply sigmoid, convert to SH0, and create empty SHN.
      - Else:
        - Use `self.splats["sh0"]` and `self.splats["shN"]` directly.
      - Export splats with means, scales, quaternions, opacities, SH0, and SHN [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def training_step(step, gaussians, cameras, gt_images, gt_depths, cfg, optimizers, schedulers, strategy, scene_scale):
    # 1. SH Degree Selection
    sh_degree_to_use = min(step // cfg.sh_degree_interval, cfg.sh_degree)
    
    # 2. Rasterization
    render_mode = 'RGB+ED' if cfg.depth_loss else 'RGB'
    renders, alphas, info = rasterize(gaussians, cameras, render_mode)
    
    # Split renders
    if renders.shape[-1] == 4:
        colors = renders[..., 0:3]
        depths = renders[..., 3:4]
    else:
        colors = renders
        depths = None
        
    # 3. Bilateral Grid Slicing
    if cfg.use_bilateral_grid:
        grid_xy = compute_grid_coordinates(cameras, info)
        colors = slice_bilateral_grid(gaussians.bil_grids, grid_xy, colors, cameras.image_ids)["rgb"]
        
    # 4. Background Compositing
    if cfg.random_bkgd:
        bkgd = torch.rand(1, 3, device=colors.device)
        colors = colors + bkgd * (1.0 - alphas)
        
    # 5. Pre-Backward Strategy
    strategy.step_pre_backward(gaussians, optimizers, step, info)
    
    # 6. Reconstruction Loss
    loss = 0.0
    if not is_nerf_sample.any():
        if cfg.use_l1_for_real_samples:
            recon_loss = F.l1_loss(colors, gt_images)
        else:
            recon_loss = F.mse_loss(colors, gt_images)
        ssim_loss = ssim_loss_function(colors, gt_images)
        loss = recon_loss * (1.0 - cfg.ssim_lambda) + ssim_loss * cfg.ssim_lambda
        
    # 7. Depth Loss
    if cfg.depth_loss and depths is not None:
        # Normalize points and sample depths
        points_normalized = normalize_points(gaussians.means)
        sampled_depths_gt = F.grid_sample(gt_depths, points_normalized)
        disp_rendered = 1.0 / (depths + 1e-6)
        disp_gt = 1.0 / (sampled_depths_gt + 1e-6)
        depth_loss = F.l1_loss(disp_rendered, disp_gt) * scene_scale
        loss += depth_loss * cfg.depth_lambda
        
    # 8. Bilateral Grid TV Loss
    if cfg.use_bilateral_grid:
        tv_loss = 10 * total_variation_loss(gaussians.bil_grids.grids)
        loss += tv_loss
        
    # 9. Opacity Regularization
    if cfg.opacity_reg > 0.0:
        opacity_reg = cfg.opacity_reg * torch.sigmoid(gaussians.opacities).mean()
        loss += opacity_reg
        
    # 10. Scale Regularization
    if cfg.scale_reg > 0.0:
        scale_reg = cfg.scale_reg * torch.exp(gaussians.scales).mean()
        loss += scale_reg
        
    # 11. Backward Pass
    loss.backward()
    
    # 12. Visibility Mask
    visibility_mask = None
    if cfg.visible_adam:
        if cfg.packed:
            visibility_mask = torch.zeros(gaussians.num_gaussians, device=colors.device)
            visibility_mask.scatter_(0, info["gaussian_ids"].flatten(), 1.0)
        else:
            visibility_mask = (info["radii"] > 0).all(-1).any(0)
            
    # 13. Optimizer Step
    if cfg.visible_adam:
        optimizers.gaussian_optimizer.step(visibility_mask)
    else:
        optimizers.gaussian_optimizer.step()
    optimizers.gaussian_optimizer.zero_grad(set_to_none=True)
    
    # 14. Pose and Bilateral Grid Optimization
    if optimizers.pose_optimizer:
        optimizers.pose_optimizer.step()
        optimizers.pose_optimizer.zero_grad(set_to_none=True)
    if optimizers.bil_grid_optimizer:
        optimizers.bil_grid_optimizer.step()
        optimizers.bil_grid_optimizer.zero_grad(set_to_none=True)
        
    # 15. Scheduler Step
    for scheduler in schedulers:
        scheduler.step()
        
    # 16. Post-Backward Strategy
    if isinstance(strategy, DefaultStrategy):
        strategy.step_post_backward(packed=cfg.packed)
    elif isinstance(strategy, MCMCStrategy):
        lr = schedulers[0].get_last_lr()[0]
        strategy.step_post_backward(lr=lr)
        
    # 17. PLY Export
    if cfg.save_ply and (step in cfg.ply_steps or step == cfg.max_steps - 1):
        if cfg.app_opt:
            sh0, shN = bake_appearance(gaussians)
        else:
            sh0 = gaussians.sh0
            shN = gaussians.shN
        export_ply(gaussians.means, gaussians.scales, gaussians.quats, gaussians.opacities, sh0, shN)
        
    return loss.item()
```

### Edge Cases and Pitfalls
- **Depth Division by Zero**: When computing disparity $1.0/\text{depths}$, add a small epsilon ($1e-6$) to avoid division by zero for pixels with zero depth [S1, unofficial].
- **SH Degree Mismatch**: Ensure that the number of SH coefficients matches the selected degree $d_{active}$. Mismatched dimensions will cause tensor shape errors during rasterization or loss computation [S1, unofficial].
- **Visibility Mask Shape**: If `cfg.packed` is true, the visibility mask must be created with size equal to the total number of Gaussians, not just the active ones. Incorrect sizing will lead to index out-of-bounds errors [S1, unofficial].
- **Random Background Consistency**: If `cfg.random_bkgd` is true, the background color should be generated per-image or per-step, not per-pixel, to maintain consistency across the image [S1, unofficial].

## Operating characteristics
- **Time Complexity**: $O(N \cdot H \cdot W)$ per step, where $N$ is the number of Gaussians, and $H, W$ are image dimensions. The rasterization step dominates.
- **Memory Complexity**: $O(N + H \cdot W)$ for storing Gaussian parameters and intermediate render buffers.
- **Performance**: Depends heavily on the number of Gaussians and the complexity of the scene. Bilateral grid and depth loss add overhead but improve convergence.
- **Strengths**: Integrates multiple loss types and post-processing steps efficiently. Supports gradual complexity increase via SH degree progression.
- **Limitations**: Requires careful tuning of regularization weights. Depth loss requires accurate ground truth depth maps.

## Background
This method is derived from the training loop implementation in AugSplat [S1, unofficial], which extends the original 3D Gaussian Splatting training procedure with additional features like bilateral grid refinement and visibility-aware Adam updates. It is commonly used in Gaussian Avatar Reconstruction and other advanced 3DGS applications.

## Sources
[S1] llazzaroni/AugSplat, https://github.com/llazzaroni/AugSplat, fetched 2026-10-27 (unofficial)
