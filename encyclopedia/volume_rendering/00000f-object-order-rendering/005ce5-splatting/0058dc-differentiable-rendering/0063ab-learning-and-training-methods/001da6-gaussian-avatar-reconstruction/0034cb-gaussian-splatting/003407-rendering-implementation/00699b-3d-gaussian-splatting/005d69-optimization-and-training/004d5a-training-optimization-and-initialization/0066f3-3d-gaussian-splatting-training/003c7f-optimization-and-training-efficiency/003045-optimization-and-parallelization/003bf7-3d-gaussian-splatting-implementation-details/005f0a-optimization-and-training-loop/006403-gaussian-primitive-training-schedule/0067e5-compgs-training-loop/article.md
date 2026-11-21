# CompGS Training Loop

Also called: CompGS Training

## Overview
The CompGS training loop is the optimization procedure for 3D Gaussian Splatting that incorporates auxiliary parameters and adaptive control mechanisms to refine Gaussian primitives. It solves the problem of static Gaussian initialization by iteratively updating positions, scales, opacities, colors, and auxiliary features through a differentiable rendering pipeline. It is distinguished from standard 3D Gaussian Splatting by the explicit management of auxiliary parameters (updated via a separate optimizer) and a specific adaptive control schedule for pruning and growing primitives based on view-dependent error metrics.

## How it works

### Inputs and Outputs
**Inputs:**
*   Dataset of training views, where each sample contains: image tensor, image dimensions ($H, W$), tangent half-FOV ($\tan(\text{fov}_x), \tan(\text{fov}_y)$), camera center, camera index, world-to-view matrix, and world-to-image projection matrix.
*   Initial Gaussian model initialized from sparse point clouds (Structure-from-Motion).
*   Configuration parameters: `max_iterations`, `ssim_weight`, `stop_iteration`, `control_base_interval`, `num_training_views`, `aux_update_enable`, `control_start_base_interval`.

**Outputs:**
*   Optimized Gaussian model parameters (means, covariances/scales, opacities, colors, auxiliary features).
*   Evaluation metrics (PSNR, SSIM) on validation views.

### Data Structures
*   **GaussianModel**: Stores parameters for $N$ Gaussians. Fields include means ($\mathbb{R}^3$), scales ($\mathbb{R}^3$), opacities ($\mathbb{R}$), colors ($\mathbb{R}^3$ or spherical harmonics coefficients), and auxiliary parameters (dimensions defined in config).
*   **RenderSettings**: Struct containing `cam_idx`, `image_height`, `image_width`, `tanfovx`, `tanfovy`, `campos`, `viewmatrix`, `projmatrix`.
*   **RenderResults**: Struct containing `rendered_img` (tensor), `predict_time`, `render_time`.
*   **Optimizers**: `WarpedAdam` instances for main Gaussian parameters and auxiliary parameters.

### Steps

1.  **Initialization**:
    *   Initialize the Gaussian model from sparse SfM point clouds.
    *   Initialize `gaussian_optimizer` and `aux_optimizer` using `WarpedAdam`.
    *   Configure learning rate schedulers for means and network parameters.

2.  **Training Loop** (for iteration $i$ from 0 to `max_iterations`):
    *   **Sample Selection**: Select a random training view sample.
    *   **Learning Rate Update**: Update learning rates for Gaussian means using the scheduler.
    *   **Rendering**:
        *   Construct `RenderSettings` from the sample.
        *   Set `retain_grad = (i < stop_iteration)`.
        *   Call `gaussian_model.render(render_settings, retain_grad)` to obtain `render_results`.
    *   **Loss Computation**:
        *   Compute L1 loss: $L_{L1} = \frac{1}{N} \sum |I_{rendered} - I_{target}|$.
        *   Compute SSIM loss: $L_{SSIM} = 1 - \text{SSIM}(I_{rendered}, I_{target})$.
        *   Compute rendering loss: $L_{render} = (1 - \text{ssim\_weight}) L_{L1} + \text{ssim\_weight} L_{SSIM}$.
        *   Compute regularization loss: $L_{reg} = 0.01 \times \text{mean}(\text{scales})$. (Note: Source describes this as "mean of the product of scales (volume proxy)", but explicitly states "0.01 times the mean of the product of scales". Implementation usually implies a volume regularization term. We follow the text: "0.01 times the mean of the product of scales").
        *   Total loss: $L_{total} = L_{render} + L_{reg}$.
    *   **Backward Pass**:
        *   If `aux_update_enable` is true and iteration is within the auxiliary update range (defined by `control_start_base_interval` and `stop_iteration`):
            *   Compute gradients for auxiliary parameters.
            *   Step `aux_optimizer`.
            *   Zero gradients for auxiliary parameters.
        *   Compute gradients for main Gaussian parameters.
        *   Step `gaussian_optimizer`.
        *   Zero gradients for main Gaussian parameters.
    *   **Adaptive Control**:
        *   If enabled and $i$ is within the control range and $i \pmod{(\text{control\_base\_interval} \times \text{num\_training\_views})} == 0$:
            *   Perform pruning (remove low-opacity Gaussians) and growing (duplicate high-error Gaussians).

3.  **Evaluation**:
    *   Render all validation views.
    *   Save original and rendered images.
    *   Calculate PSNR per view using MSE: $\text{PSNR} = 10 \log_{10}(1 / \text{MSE})$.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_compgs(dataset, gaussian_model, config):
    # 1. Initialization
    gaussian_optimizer = WarpedAdam(gaussian_model.main_params)
    aux_optimizer = WarpedAdam(gaussian_model.aux_params) if config.aux_update_enable else None
    
    lr_scheduler = partial(update_lr_means, 
                           optimizer=gaussian_optimizer, 
                           lr_update_configs=gaussian_model.get_lr_configs(config),
                           max_iterations=config.max_iterations)
    
    stop_iteration = config.adaptive_control.stop_iteration
    ssim_weight = config.training.ssim_weight
    
    for iteration in range(config.max_iterations):
        # 2. Sample and LR Update
        sample = dataset.get_random_sample()
        lr_scheduler(iteration=iteration)
        
        # 3. Rendering
        render_settings = RenderSettings(
            cam_idx=sample.cam_idx,
            image_height=sample.image_height,
            image_width=sample.image_width,
            tanfovx=sample.tan_half_fov_x,
            tanfovy=sample.tan_half_fov_y,
            campos=sample.camera_center,
            viewmatrix=sample.world_to_view_proj_mat,
            projmatrix=sample.world_to_image_proj_mat
        )
        
        retain_grad = iteration < stop_iteration
        render_results = gaussian_model.render(render_settings, retain_grad=retain_grad)
        
        # 4. Loss Computation
        l1_loss = F.l1_loss(render_results.rendered_img, sample.img)
        ssim_loss = 1 - ssim(render_results.rendered_img.unsqueeze(0), sample.img.unsqueeze(0), data_range=1.0)
        rendering_loss = (1 - ssim_weight) * l1_loss + ssim_weight * ssim_loss
        
        # Regularization: 0.01 * mean of product of scales
        # Note: 'product of scales' implies volume. If scales are (sx, sy, sz), product is sx*sy*sz.
        # The source says "mean of the product of scales".
        scales = gaussian_model.scales
        volume_proxy = torch.prod(scales, dim=-1) # Product across dimensions
        reg_loss = 0.01 * torch.mean(volume_proxy)
        
        total_loss = rendering_loss + reg_loss
        
        # 5. Backward and Optimization
        if aux_optimizer is not None:
            # Check if aux update is enabled for this iteration range
            # Source: "Update auxiliary parameters if aux_update_enable is true (iteration range defined by control_start_base_interval and stop_iteration)"
            # Assuming control_start_base_interval is a start iteration index
            if config.aux_update_enable and iteration >= config.control_start_base_interval and iteration < stop_iteration:
                total_loss.backward(retain_graph=True) # Retain graph for aux if needed, or separate backward
                # Specific implementation detail: Aux params might be updated separately
                aux_optimizer.step()
                aux_optimizer.zero_grad()
        
        total_loss.backward()
        gaussian_optimizer.step()
        gaussian_optimizer.zero_grad()
        
        # 6. Adaptive Control
        if config.adaptive_control.enabled:
            interval = config.control_base_interval * dataset.num_training_views
            if iteration < stop_iteration and iteration % interval == 0:
                adaptive_control_step(gaussian_model, iteration)

    # 7. Evaluation
    evaluate_and_save(dataset, gaussian_model, config)
```

### Edge Cases and Pitfalls
*   **Gradient Retention**: Gradients must not be retained after `stop_iteration` to save memory, but must be retained during the adaptive control phase if it occurs after the main optimization loop's gradient-free phase.
*   **Auxiliary Parameter Timing**: Auxiliary parameters are only updated within a specific iteration range. Outside this range, they remain static or are removed.
*   **Precision**: PSNR calculation involves `log10(1/MSE)`. If MSE is zero (perfect reconstruction), this results in infinity. A small epsilon should be added to MSE in practice, though not explicitly stated in the source. [G]

## Operating characteristics
*   **Time Complexity**: Dominated by the differentiable rendering step per iteration. Adaptive control adds overhead proportional to the number of primitives.
*   **Memory Complexity**: Scales with the number of Gaussians and auxiliary parameter dimensions.
*   **Performance**: Uses `WarpedAdam` optimizer. Performance metrics (PSNR/SSIM) are calculated post-training.
*   **Strengths**: Adaptive control allows dynamic refinement of the Gaussian set. Auxiliary parameters enable more complex feature representations.
*   **Limitations**: Complexity of managing two optimizers and specific iteration schedules.

## Background
The CompGS training loop is part of the CompGS (Compressed Gaussian Splatting) framework, which extends 3D Gaussian Splatting with compression and auxiliary parameter handling. It was introduced in the unofficial repository by Liu Xiangrui.

## Sources
[S1] LiuXiangrui/CompGS, https://github.com/LiuXiangrui/CompGS, fetched 2026-10-27 (unofficial)
