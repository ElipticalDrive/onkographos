# SpacetimeGaussians Training Pipeline


## Overview
The SpacetimeGaussians Training Pipeline is a specific implementation of the optimization and training loop for dynamic 3D Gaussian Splatting, designed to handle time-varying scenes by integrating motion parameters into the Gaussian representation [S1, unofficial]. It extends the standard 3D Gaussian Splatting method by introducing a densification and pruning schedule that freezes motion parameters (`_omega`) and rotations (`_rotation`) at specific iteration thresholds to stabilize learning [S1, unofficial]. The pipeline supports multiple rendering configurations (full vs. lite) and regularization modes to balance reconstruction quality and training stability [S1, unofficial].

## How it works
The pipeline manages the lifecycle of Gaussian primitives from initialization through iterative optimization, incorporating dynamic density control and parameter freezing.

### Inputs and Outputs
*   **Inputs:**
    *   `dataset`: Contains scene data, including camera poses, images, and Structure-from-Motion (SfM) points.
    *   `opt`: Configuration object containing hyperparameters such as `densify`, `reg`, `lambda_dssim`, `densify_from_iter`, `densify_until_iter`, `densification_interval`, `opacity_reset_interval`, `desicnt`, `densify_grad_threshold`, `opthr`, and `iteration` count.
    *   `gaussians`: The model object holding Gaussian parameters (position, covariance, opacity, SH coefficients, and motion parameters).
    *   `scene`: Scene manager handling camera data and loading state.
*   **Outputs:**
    *   Optimized `gaussians` model with updated parameters.
    *   Rendered images compared against ground truth for loss calculation.

### Data Structures
*   **Background Color Tensor:**
    *   Shape: `(numchannels,)`
    *   Dtype: `torch.float32`
    *   Device: `cuda`
    *   Initialization: Zeros. `numchannels` defaults to 9.
*   **Time Tensor (`gaussians.ts`):**
    *   Shape: `(1, 1, H, W)`
    *   Dtype: `torch.float32`
    *   Device: `cuda`
    *   Initialization: Ones, if `gaussians.ts` is initially `None`. `H` and `W` are derived from the first test camera.
*   **Omega Mask (`gaussians.omegamask`):**
    *   Boolean tensor used to selectively freeze motion parameters.

### Rendering Pipeline Selection
The pipeline selects a rendering backend based on the `option` parameter, retrieved via `getrenderpip(option)`:
*   `train_ours_full`: Uses `diff_gaussian_rasterization_ch9`.
*   `train_ours_lite`: Uses `diff_gaussian_rasterization_ch3`.
*   `train_ours_fullss`: Uses `diff_gaussian_rasterization_ch9`.
*   `train_ours_litess`: Uses `diff_gaussian_rasterization_ch3`.

### Training Steps
1.  **Initialization:**
    *   Initialize the Gaussian model using `getmodel(dataset.model)` with specified SH degree and RGB function.
    *   Load scene data: `Scene(dataset, gaussians, load_iteration=iteration, shuffle=False, multiview=multiview, duration=duration, loader=loader)`.
    *   Initialize `background` tensor with `numchannels` zeros on CUDA.
    *   If `gaussians.ts` is `None`, initialize it as a tensor of ones with shape `(1, 1, H, W)` on CUDA.
    *   If `gaussians.rgbdecoder` is not `None`, move it to CUDA and set to `eval` mode [G].

2.  **Iteration Loop:**
    For each training iteration:
    *   **Rendering:**
        *   Retrieve rendering pipeline and rasterizer settings via `getrenderpip(option)`.
        *   Render the scene using the selected rasterizer, passing `scaling_modifier` (default 1.0), `basicfunction` (default `trbfunction`), and other settings from `getrenderpip`.
    *   **Loss Calculation:**
        *   Compute loss $L$ based on `opt.reg`:
            *   If `reg=1`: $L = (1 - \text{opt.lambda\_dssim}) L_{L1} + \text{opt.lambda\_dssim} (1 - \text{SSIM}(I, I_{gt})) + \text{opt.regl} \frac{\sum \mathbf{g}_{\text{motion}}}{N}$
            *   If `reg=9`: $L = (1 - \text{opt.lambda\_dssim}) L_{L1} + \text{opt.lambda\_dssim} (1 - \text{SSIM}(I, I_{gt})) + \text{opt.regl} \sum_{r>0} \mathbf{g}_{\omega}^2$
            *   If `reg=10`: $L = (1 - \text{opt.lambda\_dssim}) L_{L1} + \text{opt.lambda\_dssim} (1 - \text{SSIM}(I, I_{gt})) + \text{opt.regl} \sum_{r>0} \mathbf{g}_{\text{motion}}^2$
            *   If `reg=4`: $L = (1 - \text{opt.lambda\_dssim}) L_{L1} + \text{opt.lambda\_dssim} (1 - \text{SSIM}(I, I_{gt})) + \text{opt.regl} \frac{\sum \mathbf{g}_{\text{scaling}}}{N}$
            *   If `reg=6`: $L = (1 - r) L_{L1} + r (1 - \text{SSIM}(I, I_{gt}))$ where $r = \text{clamp}(\text{mean}(I_{gt}) - 0.5 + \text{opt.lambda\_dssim}, 0, 1)$
            *   If `reg=7`: $L_{L1}$ is normalized by $2 \cdot \text{mean}(I_{gt})$.
            *   If `reg=8`: $L = (1 - \text{opt.lambda\_dssim}) L_{L1} + 0.0002 \cdot \text{opt.lambda\_dssim} \frac{\sum (\bar{x} - x)^2}{N}$
        *   Backpropagate and update parameters using Adam optimizer.
    *   **Densification and Pruning (if `densify == 1`):**
        *   **Iteration 8001:**
            *   Compute `omegamask` by calling `gaussians.zero_omegabymotion()`. This determines which Gaussians keep their motion parameters (`omega`) and which are frozen. Store in `gaussians.omegamask`.
        *   **Iterations > 8001:**
            *   Freeze weights for `_omega` using `omegamask`.
            *   Freeze weights for `_rotation` using the logical NOT of `omegamask` via `freezweightsbymasknounsqueeze`.
        *   **Densification Check:**
            *   If `iteration > opt.densify_from_iter` and `iteration % opt.densification_interval == 0`:
                *   If `flag < opt.desicnt`:
                    *   Call `gaussians.densify_pruneclone(opt.densify_grad_threshold, opt.opthr, scene.cameras_extent, size_threshold)`.
                    *   `size_threshold` is 20 if `iteration > opt.opacity_reset_interval`, else `None`.
                *   If `flag >= opt.desicnt` and `iteration < 7000`:
                    *   Prune points with opacity below `opt.opthr`.
        *   **Opacity Reset:**
            *   Every 3000 iterations, reset opacity via `gaussians.reset_opacity()`.
        *   **Final Freezing and Pruning:**
            *   If `iteration >= opt.densify_until_iter`:
                *   Freeze `_omega` and `_rotation` weights as described for iterations > 8001.
                *   If `iteration % 1000 == 500`:
                    *   Prune points where `gaussians._xyz[:,2] < 4.5`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_spacetime_gaussians(dataset, opt, gaussians, scene, iteration):
    # 1. Initialization
    numchannels = 9
    background = torch.zeros(numchannels, dtype=torch.float32, device='cuda')
    
    if gaussians.ts is None:
        H, W = scene.first_test_camera.height, scene.first_test_camera.width
        gaussians.ts = torch.ones((1, 1, H, W), dtype=torch.float32, device='cuda')
        
    if gaussians.rgbdecoder is not None:
        gaussians.rgbdecoder.to('cuda')
        gaussians.rgbdecoder.eval()

    # 2. Select Rendering Pipeline
    option = opt.render_option # e.g., 'train_ours_full'
    train_func, RasterizationSettings, Rasterizer = getrenderpip(option)
    
    scaling_modifier = 1.0
    basicfunction = trbfunction
    
    # 3. Training Loop
    for iteration in range(opt.max_iterations):
        # Render
        rendered_image = train_func(
            gaussians=gaussians,
            scene=scene,
            rasterization_settings=RasterizationSettings(
                image_height=H,
                image_width=W,
                background=background,
                scaling_modifier=scaling_modifier,
                basicfunction=basicfunction,
                GRsetting=..., # From getrenderpip
                GRzer=...     # From getrenderpip
            ),
            rasterizer=Rasterizer
        )
        
        # Loss Calculation
        gt_image = scene.get_gt_image(iteration)
        L1_loss = l1_loss(rendered_image, gt_image)
        SSIM_loss = 1 - ssim(rendered_image, gt_image)
        
        if opt.reg == 1:
            reg_term = opt.regl * (sum(g_motion) / N)
        elif opt.reg == 9:
            reg_term = opt.regl * sum(g_omega[r>0]**2)
        elif opt.reg == 10:
            reg_term = opt.regl * sum(g_motion[r>0]**2)
        elif opt.reg == 4:
            reg_term = opt.regl * (sum(g_scaling) / N)
        elif opt.reg == 6:
            r = clamp(mean(gt_image) - 0.5 + opt.lambda_dssim, 0, 1)
            L = (1 - r) * L1_loss + r * SSIM_loss
            # Backprop L
            iteration += 1
            continue
        elif opt.reg == 7:
            L1_loss = L1_loss / (2 * mean(gt_image))
            reg_term = 0
        elif opt.reg == 8:
            reg_term = 0.0002 * opt.lambda_dssim * (sum((x_bar - x)**2) / N)
        else:
            reg_term = 0
            
        L = (1 - opt.lambda_dssim) * L1_loss + opt.lambda_dssim * SSIM_loss + reg_term
        
        # Backprop
        L.backward()
        optimizer.step()
        optimizer.zero_grad()
        
        # 4. Densification and Pruning (if densify == 1)
        if opt.densify == 1:
            if iteration == 8001:
                gaussians.omegamask = gaussians.zero_omegabymotion()
                
            if iteration > 8001:
                freezweightsbymasknounsqueeze(gaussians._omega, gaussians.omegamask)
                freezweightsbymasknounsqueeze(gaussians._rotation, ~gaussians.omegamask)
                
            if iteration > opt.densify_from_iter and iteration % opt.densification_interval == 0:
                if flag < opt.desicnt:
                    size_threshold = 20 if iteration > opt.opacity_reset_interval else None
                    gaussians.densify_pruneclone(
                        opt.densify_grad_threshold,
                        opt.opthr,
                        scene.cameras_extent,
                        size_threshold
                    )
                elif iteration < 7000:
                    gaussians.prune_low_opacity(opt.opthr)
                    
            if iteration % 3000 == 0:
                gaussians.reset_opacity()
                
            if iteration >= opt.densify_until_iter:
                freezweightsbymasknounsqueeze(gaussians._omega, gaussians.omegamask)
                freezweightsbymasknounsqueeze(gaussians._rotation, ~gaussians.omegamask)
                
                if iteration % 1000 == 500:
                    gaussians.prune_z_below(4.5)
                    
        iteration += 1
```

### Parameters
*   `numchannels`: Number of channels for background color. Default: 9.
*   `scaling_modifier`: Passed to render function. Default: 1.0.
*   `basicfunction`: Passed to render function. Default: `trbfunction`.
*   `GRsetting`, `GRzer`: Rasterizer settings retrieved from `getrenderpip`.
*   `opt.reg`: Regularization mode selector.
*   `opt.lambda_dssim`: Weight for D-SSIM loss.
*   `opt.regl`: Weight for regularization term.
*   `opt.densify`: Densification mode selector (0, 1, 2, 3).
*   `opt.densify_from_iter`: Iteration to start densification.
*   `opt.densify_until_iter`: Iteration to stop densification and freeze parameters.
*   `opt.densification_interval`: Interval between densification steps.
*   `opt.opacity_reset_interval`: Interval for opacity reset.
*   `opt.desicnt`: Count threshold for densification phase.
*   `opt.densify_grad_threshold`: Gradient threshold for densification.
*   `opt.opthr`: Opacity threshold for pruning.

### Edge Cases and Pitfalls
*   **RGB Decoder:** If `gaussians.rgbdecoder` is present, it must be moved to CUDA and set to `eval` mode before rendering to prevent gradient issues or device mismatches [S1, unofficial].
*   **Pruning Logic:** Pruning behavior changes based on the densification phase. Before `opt.desicnt` is reached, densification is prioritized. After `opt.desicnt` and before iteration 7000, pruning is based on opacity. After `opt.densify_until_iter`, pruning is based on the Z-coordinate being less than 4.5, occurring every 1000 iterations at step 500 [S1, unofficial].
*   **Parameter Freezing:** The freezing of `_omega` and `_rotation` is conditional on `omegamask` and iteration count. Incorrect application can lead to unstable motion or rotation learning [S1, unofficial].

## Subtopics

### SpacetimeGaussians Densification & Pruning Schedule
A training iteration schedule that controls Gaussian densification, pruning, and parameter freezing based on iteration count, mode selection, and visibility metrics [S1, unofficial].
*   **Definition:** A structured schedule that adapts the Gaussian population density and parameter stability throughout training [S1, unofficial].
*   **Steps:**
    *   Select densification mode via `densify` parameter (0, 1, 2, or 3) [S1, unofficial].
    *   If `densify == 1`: Prune points where `gaussians._xyz[:,2] < 4.5` at iteration 1000, and reset opacity every 3000 iterations [S1, unofficial].
    *   If `densify == 3`: Update `max_radii2D` and `add_densification_stats` every iteration. Densify/prune clones at intervals. Prune low-opacity points if iteration < 7000. Reset opacity every `opt.opacity_reset_interval`. At iteration 10000, call `removeminmax` [S1, unofficial].
*   **Parameters:**
    *   `size_threshold`: Passed to `densify_pruneclone`. Set to 20 if `iteration > opt.opacity_reset_interval`, otherwise `None` [S1, unofficial].
    *   `opt.densify_until_iter`, `opt.densify_from_iter`, `opt.densification_interval`, `opt.opacity_reset_interval`, `opt.desicnt`, `opt.densify_grad_threshold`, `opt.opthr`: Control thresholds and intervals for densification, pruning, and opacity resets [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** The pipeline provides a structured approach to handling dynamic scenes by explicitly managing the lifecycle of motion parameters and Gaussian density, potentially improving stability and reconstruction quality for time-varying content [S1, unofficial].
*   **Limitations:** The rigid iteration-based schedule (e.g., freezing at 8001, pruning at 10000) may not be optimal for all scenes or dataset sizes [S1, unofficial].
*   **When to Use:** Use when training dynamic scenes with SpacetimeGaussians and requiring the specific densification and parameter freezing schedule defined in the official implementation [S1, unofficial].
*   **When Not to Use:** Avoid if a more adaptive or different densification strategy is required, or if using a different variant of Gaussian Splatting that does not support the specific rasterization backends (`diff_gaussian_rasterization_ch9`/`ch3`) [S1, unofficial].

## Background
The SpacetimeGaussians Training Pipeline originates from the paper "Spacetime Gaussian Feature Splatting for Real-Time Dynamic View Synthesis" by Li et al., presented at CVPR 2024 [S1, unofficial]. It is part of the SpacetimeGaussians project hosted by OPPO US Research [S1, unofficial]. The pipeline builds upon the 3D Gaussian Splatting method by Kerbl et al. [S1, unofficial].

## Sources
[S1] oppo-us-research/SpacetimeGaussians, https://github.com/oppo-us-research/SpacetimeGaussians, fetched 2026-10-27 (unofficial)
