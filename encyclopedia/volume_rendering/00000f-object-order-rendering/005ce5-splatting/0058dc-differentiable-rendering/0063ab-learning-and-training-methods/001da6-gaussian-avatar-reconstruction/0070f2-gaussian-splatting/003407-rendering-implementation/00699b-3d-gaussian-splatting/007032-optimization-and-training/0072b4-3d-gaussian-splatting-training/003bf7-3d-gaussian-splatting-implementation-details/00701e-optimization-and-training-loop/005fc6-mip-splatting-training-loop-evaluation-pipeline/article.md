# Mip-Splatting Training Loop & Evaluation Pipeline

Also called: 3D Gaussian Splatting Training Schedule, Mip-3DGS Training Loop

## Overview
This article defines the complete training iteration and evaluation procedure for Mip-Splatting, a method that extends 3D Gaussian Splatting by incorporating subpixel-aware rendering to mitigate aliasing artifacts [S1, unofficial]. It handles camera sampling, hybrid loss computation, adaptive densification/pruning, and quantitative metric reporting [S1, unofficial]. It differs from the standard 3D Gaussian Splatting training loop by explicitly managing subpixel offsets for both rendering and ground-truth resampling, and by adjusting the spherical harmonics degree and learning rates on a fixed schedule [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   `gaussians`: The set of 3D Gaussians $\mathcal{G} = \{g_i\}_{i=1}^N$ with parameters $\mathbf{p}_i, \mathbf{q}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_{i, l}$.
*   `viewpoint_stack`: A list of training camera poses.
*   `trainCameras`: The full set of training cameras, indexed by resolution level.
*   `gt_image`: The ground-truth image for the current camera view.
*   `pipe`: Pipeline parameters (e.g., `convert_SHs_python`, `compute_cov3D_python`).
*   `opt`: Optimization hyperparameters (learning rates, densification thresholds, intervals).
*   `dataset`: Dataset configuration flags (e.g., `ray_jitter`, `resample_gt_image`, `kernel_size`).

**Outputs:**
*   Updated `gaussians` parameters.
*   Loss value $\mathcal{L}$ for logging.
*   Quantitative metrics (SSIM, PSNR, LPIPS) during evaluation phases.

### Data Structures
*   `gaussians.max_radii2D`: A tensor of size $N$ storing the maximum image-space radius observed for each Gaussian across all previous iterations. Used for pruning [S1, unofficial].
*   `gaussians.optimizer`: An Adam optimizer instance managing gradients for all Gaussian parameters [S1, unofficial].
*   `subpixel_offset`: A tensor of shape $(H, W, 2)$ with values in $[-0.5, 0.5]$, representing the subpixel jitter for each pixel [S1, unofficial].

### Parameters
*   `--test_iterations`: List of iteration numbers for validation. Default: `[7000, 30000]` [S1, unofficial].
*   `--save_iterations`: List of iteration numbers for saving models. Default: `[7000, 30000]` [S1, unofficial].
*   `--checkpoint_iterations`: List of iteration numbers for saving optimizer checkpoints. Default: `[]` [S1, unofficial].
*   `opt.iterations`: Total training iterations. Default: 30,000 [S1, unofficial].
*   `opt.position_lr_init`: Initial position learning rate. Default: 0.00016 [S1, unofficial].
*   `opt.position_lr_final`: Final position learning rate. Default: 0.0000016 [S1, unofficial].
*   `opt.position_lr_delay_mult`: Delay multiplier for position LR. Default: 0.01 [S1, unofficial].
*   `opt.position_lr_max_steps`: Steps for position LR decay. Default: 30,000 [S1, unofficial].
*   `opt.feature_lr`: SH feature learning rate. Default: 0.0025 [S1, unofficial].
*   `opt.opacity_lr`: Opacity learning rate. Default: 0.05 [S1, unofficial].
*   `opt.scaling_lr`: Scale learning rate. Default: 0.005 [S1, unofficial].
*   `opt.rotation_lr`: Rotation learning rate. Default: 0.001 [S1, unofficial].
*   `opt.lambda_dssim`: D-SSIM weight. Default: 0.2 [S1, unofficial].
*   `opt.densification_interval`: Iterations between densification steps. Default: 100 [S1, unofficial].
*   `opt.opacity_reset_interval`: Iterations between opacity resets. Default: 3000 [S1, unofficial].
*   `opt.densify_from_iter`: Iteration when densification starts. Default: 500 [S1, unofficial].
*   `opt.densify_until_iter`: Iteration when densification stops. Default: 15,000 [S1, unofficial].
*   `opt.densify_grad_threshold`: Gradient threshold for densification. Default: 0.0002 [S1, unofficial].
*   `dataset.kernel_size`: Kernel size for rendering. Default: 0.1 [S1, unofficial].
*   `dataset.ray_jitter`: Enable subpixel jitter. Default: False [S1, unofficial].
*   `dataset.resample_gt_image`: Enable ground truth resampling. Default: False [S1, unofficial].
*   `dataset.sample_more_highres`: Prioritize high-res cameras. Default: False [S1, unofficial].
*   `dataset.white_background`: Use white background. Default: False [S1, unofficial].

### Steps

1.  **Learning Rate Update**: Update the learning rates for all Gaussian parameters based on the current iteration. This typically involves a linear decay for position learning rates from `opt.position_lr_init` to `opt.position_lr_final` over `opt.position_lr_max_steps` [S1, unofficial].
2.  **SH Degree Increment**: Every 1,000 iterations, increment the active spherical harmonics degree by one, up to the maximum defined in `dataset.sh_degree` (typically 3) [S1, unofficial].
3.  **Camera Sampling**:
    *   Pop a random camera from `viewpoint_stack`.
    *   If `dataset.sample_more_highres` is true and a random float $< 0.3$, override with a high-resolution camera from `trainCameras[highresolution_index]` [S1, unofficial].
4.  **Subpixel Offset Generation**:
    *   If `dataset.ray_jitter` is enabled, generate a tensor `subpixel_offset` of shape $(H, W, 2)$ with uniform random values in $[-0.5, 0.5]$ on CUDA.
    *   Otherwise, set `subpixel_offset = None` [S1, unofficial].
5.  **Rendering**:
    *   Render the scene using the selected camera, Gaussian parameters, and pipeline settings.
    *   Pass `kernel_size=dataset.kernel_size` and `subpixel_offset=subpixel_offset` to the rasterizer [S1, unofficial].
    *   Extract `image` (rendered color), `viewspace_point_tensor`, `visibility_filter` (boolean mask of visible Gaussians), and `radii` (image-space radii) from the render output [S1, unofficial].
6.  **Ground Truth Resampling**:
    *   If `dataset.resample_gt_image` is true, resample the ground truth image $I_{gt}$ using the `subpixel_offset` to align it with the jittered rendering grid. Let this be $I_{gt}^{resampled}$ [S1, unofficial].
    *   Otherwise, use the original $I_{gt}$ [S1, unofficial].
7.  **Loss Computation**:
    *   Compute the composite loss $\mathcal{L}$:
        $$ \mathcal{L} = (1 - \lambda_{\text{dssim}}) \mathcal{L}_{\text{L1}} + \lambda_{\text{dssim}} (1 - \text{SSIM}(I_{\text{render}}, I_{gt}^{resampled})) $$
        where $\mathcal{L}_{\text{L1}}$ is the mean absolute error between $I_{\text{render}}$ and $I_{gt}^{resampled}$, and $\lambda_{\text{dssim}} = 0.2$ [S1, unofficial].
8.  **Statistics Tracking**:
    *   Update `gaussians.max_radii2D`: For visible Gaussians (`visibility_filter`), update the maximum observed radius:
        $$ \text{max\_radii2D}[i] = \max(\text{max\_radii2D}[i], \text{radii}[i]) \quad \forall i \in \text{visibility\_filter} $$
    *   Add densification statistics: Accumulate gradient norms and other metrics for the visible Gaussians using `viewspace_point_tensor` and `visibility_filter` [S1, unofficial].
9.  **Densification and Pruning** (Conditional):
    *   If `iteration > opt.densify_from_iter` and `iteration % opt.densification_interval == 0`:
        *   Determine `size_threshold`: If `iteration > opt.opacity_reset_interval`, set `size_threshold = 20`; otherwise, `size_threshold = None` [S1, unofficial].
        *   Call `gaussians.densify_and_prune(opt.densify_grad_threshold, 0.005, scene.cameras_extent, size_threshold)` to split, clone, or remove Gaussians based on gradient magnitude, opacity, and size [S1, unofficial].
        *   Call `gaussians.compute_3D_filter(cameras=trainCameras)` to update the 3D covariance filters for the modified Gaussian set [S1, unofficial].
10. **Opacity Reset** (Conditional):
    *   If `iteration % opt.opacity_reset_interval == 0` or (`dataset.white_background` is true and `iteration == opt.densify_from_iter`), reset the opacities of all Gaussians to a low initial value (e.g., 0.01) to prevent early saturation [S1, unofficial].
11. **3D Filter Update** (Post-Densification):
    *   If `iteration > opt.densify_until_iter` and `iteration % 100 == 0` and `iteration < opt.iterations - 100`, call `gaussians.compute_3D_filter(cameras=trainCameras)` to maintain accurate covariance representations after densification has ceased [S1, unofficial].
12. **Optimization Step**:
    *   Perform the optimizer step: `gaussians.optimizer.step()` [S1, unofficial].
    *   Zero gradients: `gaussians.optimizer.zero_grad(set_to_none=True)` [S1, unofficial].
13. **Evaluation** (Conditional):
    *   If `iteration` is in `--test_iterations`:
        *   Render the scene for all test cameras.
        *   Load rendered images from `test_preds_{scale}` and ground truth from `gt_{scale}` [S1, unofficial].
        *   Compute per-image SSIM, PSNR, and LPIPS (using a VGG network) [S1, unofficial].
        *   Average metrics across all test images.
        *   Log to TensorBoard and save to `results.json` and `per_view.json` [S1, unofficial].
14. **Checkpointing** (Conditional):
    *   If `iteration` is in `--save_iterations`, save the Gaussian model state [S1, unofficial].
    *   If `iteration` is in `--checkpoint_iterations`, save the optimizer state [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mip_splatting_training_loop(gaussians, viewpoint_stack, trainCameras, dataset, opt, pipe, iteration):
    """
    One iteration of the Mip-Splatting training loop.
    """
    # 1. Update Learning Rates
    gaussians.update_learning_rate(iteration)
    
    # 2. Increment SH Degree
    if iteration % 1000 == 0:
        gaussians.oneupSHdegree()
    
    # 3. Sample Camera
    if dataset.sample_more_highres and random.random() < 0.3:
        viewpoint_cam = trainCameras[highresolution_index]
    else:
        viewpoint_cam = viewpoint_stack.pop()
    
    # 4. Generate Subpixel Offset
    if dataset.ray_jitter:
        subpixel_offset = torch.rand(H, W, 2, device='cuda') * 2.0 - 1.0  # Values in [-0.5, 0.5]
    else:
        subpixel_offset = None
    
    # 5. Render Scene
    render_dict = render(
        viewpoint_cam, 
        gaussians, 
        pipe, 
        background, 
        kernel_size=dataset.kernel_size, 
        subpixel_offset=subpixel_offset
    )
    image = render_dict['render']
    viewspace_point_tensor = render_dict['viewspace_points']
    visibility_filter = render_dict['visibility_filter']
    radii = render_dict['radii']
    
    # 6. Resample Ground Truth
    if dataset.resample_gt_image and subpixel_offset is not None:
        gt_image = create_offset_gt(gt_image, subpixel_offset)
    else:
        gt_image = gt_image
    
    # 7. Compute Loss
    l1_loss = (image - gt_image).abs().mean()
    ssim_loss = 1 - ssim(image, gt_image)
    loss = (1 - opt.lambda_dssim) * l1_loss + opt.lambda_dssim * ssim_loss
    
    # 8. Track Statistics
    gaussians.max_radii2D[visibility_filter] = torch.max(
        gaussians.max_radii2D[visibility_filter], 
        radii[visibility_filter]
    )
    gaussians.add_densification_stats(viewspace_point_tensor, visibility_filter)
    
    # 9. Densification and Pruning
    if iteration > opt.densify_from_iter and iteration % opt.densification_interval == 0:
        size_threshold = 20 if iteration > opt.opacity_reset_interval else None
        gaussians.densify_and_prune(
            opt.densify_grad_threshold, 
            0.005, 
            scene.cameras_extent, 
            size_threshold
        )
        gaussians.compute_3D_filter(cameras=trainCameras)
    
    # 10. Opacity Reset
    if iteration % opt.opacity_reset_interval == 0:
        gaussians.reset_opacities()
    elif dataset.white_background and iteration == opt.densify_from_iter:
        gaussians.reset_opacities()
    
    # 11. 3D Filter Update (Post-Densification)
    if iteration > opt.densify_until_iter and iteration % 100 == 0 and iteration < opt.iterations - 100:
        gaussians.compute_3D_filter(cameras=trainCameras)
    
    # 12. Optimization Step
    loss.backward()
    gaussians.optimizer.step()
    gaussians.optimizer.zero_grad(set_to_none=True)
    
    # 13. Evaluation (if applicable)
    if iteration in opt.test_iterations:
        evaluate_and_log(gaussians, trainCameras, iteration)
    
    # 14. Checkpointing (if applicable)
    if iteration in opt.save_iterations:
        save_gaussian_model(gaussians, iteration)
    if iteration in opt.checkpoint_iterations:
        save_optimizer_checkpoint(gaussians.optimizer, iteration)
    
    return loss.item()
```

### Edge Cases and Pitfalls
*   **Precision**: Subpixel jitter requires floating-point precision for offset generation and resampling. Ensure `subpixel_offset` is on the same device as the image tensors [S1, unofficial].
*   **Boundaries**: When resampling ground truth images, handle boundary pixels correctly to avoid artifacts from interpolation [S1, unofficial].
*   **Degenerate Input**: If `visibility_filter` is empty (no visible Gaussians), skip densification statistics and loss computation to avoid division by zero or invalid operations [G].
*   **SH Degree**: Ensure SH degree does not exceed the maximum supported by the renderer or the data [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
*   **Time**: Dominated by the rasterization step, which is $O(N \cdot K)$ where $N$ is the number of Gaussians and $K$ is the kernel size. Densification adds $O(N)$ overhead every 100 iterations.
*   **Memory**: $O(N)$ for Gaussian parameters and $O(H \cdot W)$ for intermediate render buffers. `max_radii2D` adds $O(N)$ memory [S1, unofficial].

### Measured Performance
*   Performance metrics (SSIM, PSNR, LPIPS) are reported at iterations 7,000 and 30,000 [S1, unofficial].
*   Specific performance numbers depend on the dataset and hardware. The method is designed to improve visual quality over standard 3DGS by reducing aliasing [S1, unofficial].

### Scaling
*   Scales with the number of Gaussians $N$ and image resolution $H \times W$.
*   High-resolution training can be prioritized via `dataset.sample_more_highres` [S1, unofficial].

### Strengths and Limitations
*   **Strengths**: Reduces aliasing artifacts through subpixel-aware rendering and ground truth resampling. Adaptive densification ensures efficient representation [S1, unofficial].
*   **Limitations**: Increased computational cost due to subpixel jitter and resampling. Requires careful tuning of jitter and resampling flags [S1, unofficial].

### When to Use
*   Use when high-fidelity rendering with minimal aliasing is required.
*   Use when training on high-resolution datasets where standard 3DGS may exhibit jagged edges [S1, unofficial].

### When Not to Use
*   Avoid if computational resources are severely limited and aliasing is acceptable.
*   Avoid if ground truth images are not available for resampling [S1, unofficial].

## Background

### Origin
Mip-Splatting was introduced by the autonomousvision group, extending the original 3D Gaussian Splatting method [S1, unofficial]. The codebase is available on GitHub [S1, unofficial].

### History and Development
The method builds upon the differentiable rendering framework of 3D Gaussian Splatting, adding subpixel rendering capabilities to address aliasing issues [S1, unofficial].

### Where It Is Used
Used in 3D reconstruction and novel view synthesis tasks where high visual fidelity is critical [S1, unofficial].

## Sources

[S1] autonomousvision/mip-splatting, https://github.com/autonomousvision/mip-splatting, fetched 2026-10-26 (unofficial)
