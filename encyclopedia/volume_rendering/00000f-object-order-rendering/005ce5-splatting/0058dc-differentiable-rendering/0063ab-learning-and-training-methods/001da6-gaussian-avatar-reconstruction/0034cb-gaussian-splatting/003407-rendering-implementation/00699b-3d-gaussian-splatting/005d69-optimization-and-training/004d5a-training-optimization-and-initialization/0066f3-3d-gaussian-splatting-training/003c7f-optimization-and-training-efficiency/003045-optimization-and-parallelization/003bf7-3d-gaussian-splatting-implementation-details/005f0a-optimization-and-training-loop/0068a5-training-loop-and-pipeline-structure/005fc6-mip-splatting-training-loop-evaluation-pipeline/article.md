# Mip-Splatting Training Loop & Evaluation Pipeline

Also called: 3D Gaussian Splatting Training Schedule, Mip-3DGS Training Loop

## Overview
This method defines the complete training iteration and evaluation procedure for Mip-Splatting, a variant of 3D Gaussian Splatting that addresses aliasing by rendering at multiple resolutions [S1, unofficial]. It handles camera sampling, subpixel-aware rendering, hybrid loss computation, adaptive densification/pruning, and quantitative metric reporting [S1, unofficial]. It distinguishes itself from the standard [3D Gaussian Splatting Training Loop](15139-3d-gaussian-splatting-training-loop.md) by introducing subpixel jittering for both the rendered image and the ground truth, and by dynamically adjusting the spherical harmonics (SH) degree during training to prevent high-frequency artifacts [S1, unofficial].

## How it works
**Inputs:**
*   `gaussians`: The set of 3D Gaussians with parameters (position, covariance, opacity, SH coefficients).
*   `viewpoint_stack`: A list of available training cameras.
*   `trainCameras`: The full list of training cameras, indexed by resolution level.
*   `gt_image`: The ground truth image for the sampled view.
*   `opt`: Optimization hyperparameters (learning rates, densification thresholds, intervals).
*   `dataset`: Dataset configuration flags (`ray_jitter`, `resample_gt_image`, `sample_more_highres`, `kernel_size`).

**Outputs:**
*   Updated `gaussians` parameters.
*   Quantitative metrics (SSIM, PSNR, LPIPS) during evaluation phases.

**Data Structures:**
*   `gaussians.max_radii2D`: A tensor tracking the maximum image-space radius for each Gaussian, used for pruning.
*   `gaussians.viewspace_point_tensor`: Stores the 2D screen-space positions of Gaussians for gradient-based densification.

**Parameters:**
*   `opt.iterations`: Total training iterations (Default: 30,000) [S1, unofficial].
*   `opt.densify_from_iter`: Iteration after which densification begins (Default: 500) [S1, unofficial].
*   `opt.densify_until_iter`: Iteration after which densification stops (Default: 15,000) [S1, unofficial].
*   `opt.densification_interval`: Frequency of densification steps (Default: 100) [S1, unofficial].
*   `opt.opacity_reset_interval`: Frequency of opacity resets (Default: 3,000) [S1, unofficial].
*   `opt.densify_grad_threshold`: Gradient threshold for splitting (Default: 0.0002) [S1, unofficial].
*   `opt.lambda_dssim`: Weight for the D-SSIM loss term (Default: 0.2) [S1, unofficial].
*   `dataset.kernel_size`: Kernel size for rendering (Default: 0.1) [S1, unofficial].
*   `dataset.ray_jitter`: Boolean flag to enable subpixel offset generation [S1, unofficial].
*   `dataset.resample_gt_image`: Boolean flag to resample ground truth images [S1, unofficial].
*   `dataset.sample_more_highres`: Boolean flag to increase high-resolution sampling probability [S1, unofficial].

**Steps:**

1.  **Learning Rate Update:** Update the learning rates for all Gaussian parameters using `gaussians.update_learning_rate(iteration)` [S1, unofficial].
2.  **SH Degree Increase:** Every 1,000 iterations, increment the active spherical harmonics degree using `gaussians.oneupSHdegree()` [S1, unofficial].
3.  **Camera Sampling:** Pop a random camera from `viewpoint_stack`. If `dataset.sample_more_highres` is true and a random float is less than 0.3, override the selection with a high-resolution camera from `trainCameras[highresolution_index]` [S1, unofficial].
4.  **Subpixel Offset Generation:** If `dataset.ray_jitter` is enabled, generate a subpixel offset tensor of shape `(H, W, 2)` with values uniformly distributed in `[-0.5, 0.5]` on CUDA. Otherwise, set `subpixel_offset = None` [S1, unofficial].
5.  **Rendering:** Render the scene by calling `render(viewpoint_cam, gaussians, pipe, background, kernel_size=dataset.kernel_size, subpixel_offset=subpixel_offset)` [S1, unofficial]. Extract the rendered `image`, `viewspace_point_tensor`, `visibility_filter`, and `radii` from the returned dictionary [S1, unofficial].
6.  **Ground Truth Resampling:** If `dataset.resample_gt_image` is true, resample the ground truth image using the `subpixel_offset` via `create_offset_gt(gt_image, subpixel_offset)` [S1, unofficial].
7.  **Statistics Tracking:**
    *   Update max image-space radii for pruning: `gaussians.max_radii2D[visibility_filter] = torch.max(gaussians.max_radii2D[visibility_filter], radii[visibility_filter])` [S1, unofficial].
    *   Add densification statistics: `gaussians.add_densification_stats(viewspace_point_tensor, visibility_filter)` [S1, unofficial].
8.  **Densification and Pruning:** If `iteration > opt.densify_from_iter` and `iteration % opt.densification_interval == 0`:
    *   Determine `size_threshold`: Set to 20 if `iteration > opt.opacity_reset_interval`, else `None` [S1, unofficial].
    *   Call `gaussians.densify_and_prune(opt.densify_grad_threshold, 0.005, scene.cameras_extent, size_threshold)` [S1, unofficial].
    *   Call `gaussians.compute_3D_filter(cameras=trainCameras)` [S1, unofficial].
9.  **Opacity Reset:** If `iteration % opt.opacity_reset_interval == 0` or if `dataset.white_background` is true and `iteration == opt.densify_from_iter`, reset the opacities of the Gaussians [S1, unofficial].
10. **3D Filter Update (Post-Densification):** If `iteration > opt.densify_until_iter`, if `iteration % 100 == 0`, and if `iteration < opt.iterations - 100`, update the 3D filter via `gaussians.compute_3D_filter(cameras=trainCameras)` [S1, unofficial].
11. **Optimizer Step:** Perform the optimizer step using `gaussians.optimizer.step()` followed by `gaussians.optimizer.zero_grad(set_to_none=True)` [S1, unofficial].

**Loss Function:**
The total loss $\mathcal{L}$ is computed as:
$$ \mathcal{L} = (1 - \lambda_{\text{dssim}}) \mathcal{L}_{\text{L1}} + \lambda_{\text{dssim}} (1 - \text{SSIM}(I_{\text{render}}, I_{\text{gt}})) $$
where $\mathcal{L}_{\text{L1}}$ is the L1 loss between the rendered image $I_{\text{render}}$ and the (potentially resampled) ground truth image $I_{\text{gt}}$, and $\lambda_{\text{dssim}}$ is the D-SSIM weight [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mip_splatting_training_loop(iteration, gaussians, viewpoint_stack, trainCameras, gt_image, opt, dataset, scene):
    # 1. Update Learning Rates
    gaussians.update_learning_rate(iteration)

    # 2. Increase SH Degree every 1000 iterations
    if iteration % 1000 == 0:
        gaussians.oneupSHdegree()

    # 3. Sample Camera
    viewpoint_cam = viewpoint_stack.pop()
    if dataset.sample_more_highres and random.random() < 0.3:
        viewpoint_cam = trainCameras[highresolution_index]

    # 4. Generate Subpixel Offset
    if dataset.ray_jitter:
        subpixel_offset = torch.rand((viewpoint_cam.image_height, viewpoint_cam.image_width, 2), device='cuda') - 0.5
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
    image = render_dict["render"]
    viewspace_point_tensor = render_dict["viewspace_points"]
    visibility_filter = render_dict["visibility_filter"]
    radii = render_dict["radii"]

    # 6. Resample Ground Truth if enabled
    if dataset.resample_gt_image:
        gt_image_resampled = create_offset_gt(gt_image, subpixel_offset)
    else:
        gt_image_resampled = gt_image

    # 7. Track Statistics
    gaussians.max_radii2D[visibility_filter] = torch.max(
        gaussians.max_radii2D[visibility_filter], 
        radii[visibility_filter]
    )
    gaussians.add_densification_stats(viewspace_point_tensor, visibility_filter)

    # 8. Densification and Pruning
    if iteration > opt.densify_from_iter and iteration % opt.densification_interval == 0:
        size_threshold = 20 if iteration > opt.opacity_reset_interval else None
        gaussians.densify_and_prune(
            opt.densify_grad_threshold, 
            0.005, 
            scene.cameras_extent, 
            size_threshold
        )
        gaussians.compute_3D_filter(cameras=trainCameras)

    # 9. Opacity Reset
    if iteration % opt.opacity_reset_interval == 0:
        gaussians.reset_opacities()
    elif dataset.white_background and iteration == opt.densify_from_iter:
        gaussians.reset_opacities()

    # 10. Post-Densification 3D Filter Update
    if iteration > opt.densify_until_iter:
        if iteration % 100 == 0 and iteration < opt.iterations - 100:
            gaussians.compute_3D_filter(cameras=trainCameras)

    # 11. Optimizer Step
    gaussians.optimizer.step()
    gaussians.optimizer.zero_grad(set_to_none=True)

    return image, gt_image_resampled
```

**Evaluation Pipeline:**
During evaluation phases (triggered by `--test_iterations`):
1.  Load rendered images from `test_preds_{scale}` and ground truth images from `gt_{scale}` directories [S1, unofficial].
2.  Compute per-image SSIM, PSNR, and LPIPS (using a VGG network) [S1, unofficial].
3.  Average metrics across all images in the view set [S1, unofficial].
4.  Log metrics to TensorBoard and save to `results.json` and `per_view.json` [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The training loop adds overhead for subpixel offset generation and ground truth resampling, but the core rasterization complexity remains similar to standard 3DGS. The periodic `compute_3D_filter` calls add computational cost, particularly after densification stops [S1, unofficial].
*   **Memory Complexity:** Requires storing `max_radii2D` and `viewspace_point_tensor` for each Gaussian. The subpixel offset tensor requires $O(H \times W \times 2)$ memory per iteration [S1, unofficial].
*   **Strengths:** Reduces aliasing artifacts through subpixel jittering and multi-resolution rendering. The adaptive SH degree increase prevents high-frequency noise in early training stages [S1, unofficial].
*   **Limitations:** The subpixel jittering and resampling require careful implementation to maintain differentiability. The periodic 3D filter updates can slow down the latter stages of training if not optimized [S1, unofficial].
*   **When to use:** When rendering scenes with high-frequency details or when aliasing artifacts are prominent in standard 3DGS.
*   **When not to use:** When computational resources are extremely limited and the overhead of subpixel operations is prohibitive.

## Background
Mip-Splatting was introduced by the autonomousvision group as an extension to 3D Gaussian Splatting to address aliasing issues [S1, unofficial]. It builds upon the [3D Gaussian Splatting](15139-3d-gaussian-splatting-training-loop.md) framework by incorporating techniques from mip-mapping and subpixel rendering [S1, unofficial]. The method is implemented in the `mip-splatting` repository [S1, unofficial].

## Sources
[S1] autonomousvision/mip-splatting, https://github.com/autonomousvision/mip-splatting, fetched 2026-10-26 (unofficial)
