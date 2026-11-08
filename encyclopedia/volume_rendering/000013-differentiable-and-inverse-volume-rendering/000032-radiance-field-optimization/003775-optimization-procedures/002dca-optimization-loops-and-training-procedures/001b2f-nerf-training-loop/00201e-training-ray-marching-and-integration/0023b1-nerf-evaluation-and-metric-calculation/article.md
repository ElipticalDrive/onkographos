# NeRF Evaluation and Metric Calculation

Also called: Evaluation and Metric Calculation, NeRF Evaluation, NeRF Validation, Single Image Rendering, Validation Rendering

## Overview
NeRF Evaluation and Metric Calculation renders a single image from a trained NeRF model at a specified camera pose and computes quantitative performance metrics (PSNR, SSIM, LPIPS) by comparing the rendered output against ground truth data [S1, S2, S3, S4, unofficial]. It differs from training ray marching by disabling stochastic perturbations to ensure deterministic, reproducible results and by often handling background compositing explicitly for fair comparison [S2, S4, unofficial]. It serves as the validation step in the optimization loop, providing the scalar losses and visual quality indicators required for monitoring convergence [S3, unofficial].

## How it works
**Inputs:**
*   `coarse_model`, `fine_model`: Trained NeRF MLPs [S1, unofficial].
*   `pose`: 4x4 camera-to-world matrix (numpy array or torch tensor) [S1, S2, unofficial].
*   `K`: 3x3 camera intrinsic matrix (numpy array or torch tensor) [S1, unofficial].
*   `H`, `W`: Image height and width in pixels [S1, S2, unofficial].
*   `near`, `far`: Near and far plane distances for ray marching [S1, unofficial].
*   `gt_image`: Ground truth RGB image (H x W x 3) [S2, S3, unofficial].
*   `args`: Configuration object containing `chunk` (default 1024), `n_samples_coarse`, `n_samples_fine`, `white_bkgd`, `lindisp`, `netchunk`, `use_ndc` [S1, unofficial].
*   `device`: Torch device (CPU/GPU) [S1, unofficial].

**Outputs:**
*   `rgb`: Rendered RGB image (H x W x 3) [S1, unofficial].
*   `depth`: Depth map (H x W) [S1, unofficial].
*   `disp`: Disparity map (H x W) [S1, unofficial].
*   `metrics`: Dictionary containing PSNR, SSIM, and optionally LPIPS values [S3, S4, unofficial].

**Data Structures:**
*   `rays_o`: Tensor of shape `[H*W, 3]` containing ray origins [S1, unofficial].
*   `rays_d`: Tensor of shape `[H*W, 3]` containing ray directions [S1, unofficial].
*   `render_results`: Dictionary mapping keys `'rgb'`, `'depth'`, `'disp'` to tensors of shape `[H, W, 3]` or `[H, W]` [S1, unofficial].

**Steps:**
1.  **Preparation:** Set `coarse_model` and `fine_model` to evaluation mode (`model.eval()`) [S2, S3, unofficial]. Convert `pose` and `K` to torch tensors of type `float32` on the specified `device` if they are numpy arrays [S1, unofficial].
2.  **Ray Generation:** Generate ray origins `rays_o` and directions `rays_d` using the intrinsic matrix `K`, image dimensions `H`, `W`, and camera pose `pose` [S1, S3, unofficial]. Flatten `rays_o` and `rays_d` to shape `[-1, 3]` [S1, unofficial].
3.  **NDC Conversion (Optional):** If `args.use_ndc` is true, convert `rays_o` and `rays_d` to Normalized Device Coordinates using the focal length extracted from `K` and a near plane of 1.0 [S1, unofficial].
4.  **Chunked Rendering:** Process the flattened rays in chunks of size `args.chunk` (default 1024) to manage memory [S2, unofficial]. For each chunk:
    *   Call the rendering function (e.g., `batchify_render` or `render_rays`) with `rays_o_chunk`, `rays_d_chunk`, `coarse_model`, `fine_model`, `near`, `far`, `args.n_samples_coarse`, `args.n_samples_fine`, `perturb=False`, `raw_noise_std=0.0`, `args.white_bkgd`, `args.lindisp`, and `args.netchunk` [S1, S2, unofficial].
    *   *Note:* Perturbation is explicitly disabled (`perturb=False`) to ensure deterministic results [S2, unofficial].
5.  **Aggregation:** Concatenate the results from all chunks [S2, unofficial]. Determine the keys for RGB, depth, and disparity based on the presence of `'_fine'` in the result keys, falling back to `'_coarse'` if necessary [S1, unofficial].
6.  **Reshaping:** Reshape `result[rgb_key]` to `[H, W, 3]`, `result[depth_key]` to `[H, W]`, and `result[disp_key]` to `[H, W]` [S1, unofficial]. Move tensors to CPU and convert to numpy arrays [S1, unofficial].
7.  **Metric Calculation:**
    *   **MSE:** Calculate Mean Squared Error between predicted RGB and `gt_image` [S3, unofficial].
    *   **PSNR:** Compute $PSNR = -10 \cdot \log_{10}(MSE)$ [S3, unofficial].
    *   **SSIM:** Compute Structural Similarity Index using `compare_ssim` with `full=True` and `multichannel=True` [S3, unofficial].
    *   **LPIPS (Optional):** If supported, compute Learned Perceptual Image Patch Similarity using a VGG network module [S4, unofficial].
8.  **Output:** Return `rgb`, `depth`, `disp`, and the computed metrics [S1, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from torch.nn.functional import mse_loss

def evaluate_nerf(coarse_model, fine_model, pose, K, H, W, near, far, gt_image, args, device):
    """
    Render a single image and compute evaluation metrics.
    """
    # 1. Set models to eval mode
    coarse_model.eval()
    fine_model.eval()
    
    # 2. Convert inputs to tensors
    if isinstance(pose, np.ndarray):
        pose = torch.from_numpy(pose).float().to(device)
    if isinstance(K, np.ndarray):
        K = torch.from_numpy(K).float().to(device)
        
    # 3. Generate rays
    # Assuming get_rays is defined in parent context or utility
    rays_o, rays_d = get_rays(H, W, K, pose)
    
    # 4. NDC conversion if enabled
    if args.use_ndc:
        focal = K[0, 0]
        rays_o, rays_d = ndc_rays(H, W, focal, 1.0, rays_o, rays_d)
        
    # Flatten rays
    rays_o = rays_o.reshape(-1, 3)
    rays_d = rays_d.reshape(-1, 3)
    
    # 5. Chunked rendering
    chunk = args.chunk
    all_rgb = []
    all_depth = []
    all_disp = []
    
    for i in range(0, rays_o.shape[0], chunk):
        rays_o_chunk = rays_o[i:i+chunk]
        rays_d_chunk = rays_d[i:i+chunk]
        
        # Call batchify_render with perturb=False
        # This function handles coarse/fine sampling and integration
        result = batchify_render(
            rays_o_chunk, rays_d_chunk, chunk,
            coarse_model, fine_model,
            near, far,
            args.n_samples_coarse, args.n_samples_fine,
            perturb=False, raw_noise_std=0.0,
            white_bkgd=args.white_bkgd, lindisp=args.lindisp,
            netchunk=args.netchunk
        )
        
        # Determine keys
        rgb_key = 'rgb_fine' if 'rgb_fine' in result else 'rgb_coarse'
        depth_key = 'depth_fine' if 'depth_fine' in result else 'depth_coarse'
        disp_key = 'disp_fine' if 'disp_fine' in result else 'disp_coarse'
        
        # Reshape and collect
        rgb_chunk = result[rgb_key].reshape(-1, 3)
        depth_chunk = result[depth_key].reshape(-1)
        disp_chunk = result[disp_key].reshape(-1)
        
        all_rgb.append(rgb_chunk)
        all_depth.append(depth_chunk)
        all_disp.append(disp_chunk)
        
    # Concatenate results
    rgb_flat = torch.cat(all_rgb, dim=0)
    depth_flat = torch.cat(all_depth, dim=0)
    disp_flat = torch.cat(all_disp, dim=0)
    
    # Reshape to image dimensions
    rgb_image = rgb_flat.reshape(H, W, 3).cpu().numpy()
    depth_map = depth_flat.reshape(H, W).cpu().numpy()
    disp_map = disp_flat.reshape(H, W).cpu().numpy()
    
    # 6. Metric Calculation
    # Convert GT to tensor if needed
    gt_tensor = torch.from_numpy(gt_image).float().to(device)
    pred_tensor = torch.from_numpy(rgb_image).float().to(device)
    
    # MSE
    mse = mse_loss(pred_tensor, gt_tensor)
    
    # PSNR
    psnr = -10.0 * torch.log10(mse).item()
    
    # SSIM (using a standard library function)
    # Note: compare_ssim expects float32 tensors or arrays
    ssim = compare_ssim(rgb_image, gt_image, full=True, multichannel=True)
    
    # LPIPS (if available)
    lpips_val = 0.0
    if hasattr(args, 'use_lpips') and args.use_lpips:
        lpips_val = compute_lpips(pred_tensor, gt_tensor)
        
    metrics = {
        'psnr': psnr,
        'ssim': ssim,
        'lpips': lpips_val
    }
    
    return rgb_image, depth_map, disp_map, metrics
```

**Parameters:**
*   `chunk`: Integer, default 1024. Number of rays processed per batch to limit memory usage [S2, unofficial].
*   `perturb`: Boolean, default `False`. Enables/disables jittering of sample points along rays [S2, unofficial].
*   `white_bkgd`: Boolean. If true, composites the result against a white background [S1, unofficial].
*   `lindisp`: Boolean. If true, samples linearly in disparity space rather than depth space [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Perturbation:** Must be disabled (`perturb=False`) during evaluation to ensure deterministic results [S2, unofficial].
*   **Background:** If the ground truth has an alpha channel or transparent background, explicit compositing (e.g., against white or black) is required for fair PSNR/SSIM comparison [S2, unofficial].
*   **Precision:** Ensure inputs are `float32` to avoid precision issues in ray generation and rendering [S1, unofficial].
*   **Memory:** Large images or high sample counts may exceed GPU memory; chunking is essential [S2, unofficial].

## Subtopics

### [Unified Metric Aggregation](3644-unified-metric-aggregation.md)
Unified Metric Aggregation processes the outputs of the NeRF Evaluation and Metric Calculation stage to produce standardized quantitative performance reports. It aggregates per-view metrics (PSNR, SSIM, LPIPS) into global averages and per-view dictionaries, enabling consistent comparison across different training iterations and datasets. Unlike raw metric computation, this method handles the orchestration of multiple metric types, data normalization, and structured output generation (JSON/text). It supports different metric styles (e.g., `standard_3dgs` vs `beta_splatting`) and validates that all metric values are finite before saving.

### [PSNR Evaluation Pipeline](4315-psnr-evaluation-pipeline.md)
The PSNR Evaluation Pipeline computes the Peak Signal-to-Noise Ratio (PSNR) on held-out test or validation splits to quantify the rendering quality of a trained NeRF model. It differs from training metrics by using inference-optimized rendering paths and explicitly handling background compositing to ensure fair comparison against ground truth images. It calculates MSE and derives PSNR using $PSNR = -10 \cdot \log_{10}(MSE)$.

### [Masked Evaluation](7531-masked-evaluation.md)
A procedure to restrict radiance field evaluation to pixels visible in at least two training views, preventing metric degradation from predicting unseen geometry or depth of single-view objects. This ensures that metrics reflect the model's ability to reconstruct observed geometry rather than hallucinating details in occluded regions.

### Official Metrics Evaluation
A variant that loads render and ground truth images as tensors, handling batched or unbatched formats [S4, unofficial]. It selects metric functions based on style (e.g., `standard_3dgs` uses `loss_utils.ssim` with `batched=True`; `beta_splatting` uses `fused_ssim` with `batched=False`) [S4, unofficial]. It computes LPIPS using `lpips_module.lpips(render, gt, net_type="vgg").mean()` and checks for finite values before computing mean metrics across all views and saving to `results.json` and `per_view.json` [S4, unofficial].

### Depth Supervision for TensoRF
TensoRF augmented with depth supervision derived from COLMAP keypoints to improve low-data performance [S5, unofficial]. It extracts 3D point locations and 2D pixel coordinates of COLMAP keypoints, calculates rays $r(t) = o + td$ and depth $D$ from the projection origin, and samples the model to calculate estimated depths $\hat{D}$ [S5, unofficial]. The depth loss is computed as $L_{\hat{D}} = \mathbb{E}_{r \in R(P)} ||\hat{D}(r) - D(r)||_2^2$ [S5, unofficial]. This is typically combined with the color loss $L_{tot} = L_{\hat{C}} + \lambda_i L_{\hat{D}}$ [S5, unofficial]. It is often evaluated on datasets like LLFF (20-30 images) [S5, unofficial].

### LiDAR Semantic Filtering for Evaluation
A procedure to evaluate geometry reconstruction quality using LiDAR data [S6]. It identifies LiDAR points within the camera's field of view and selects rays terminating on desired semantic classes (e.g., roads, sidewalks) using segmentation predictions (e.g., SegFormer) [S6]. These filtered LiDAR rays are fed into the NeRF model for depth rendering, and the predicted point clouds are accumulated and compared against ground truth LiDAR point clouds for each test frame [S6]. Plane reconstruction quality is assessed using Plane Standard Deviation ($P\sigma$) calculated via RANSAC on 3x3 square meter patches [S6].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of pixels ($H \times W$) and the number of samples per ray ($N$). Chunking adds overhead but prevents memory overflow [S2, unofficial].
*   **Memory Complexity:** Dominated by the chunk size. With `chunk=1024`, memory usage is proportional to $1024 \times N \times \text{model\_size}$ [S2, unofficial].
*   **Performance:** Rendering a single image is significantly faster than training a step, but still costly for high-resolution images or large scenes [S1, S2, unofficial].
*   **Strengths:** Deterministic results (with `perturb=False`), modular metric calculation, supports various loss functions (MSE, SSIM, LPIPS, Depth) [S2, S3, S4, S5, unofficial].
*   **Limitations:** Computationally expensive for large datasets; requires careful handling of background and occlusions for fair metrics [S2, unofficial][S6].
*   **When to use:** During validation phases, hyperparameter tuning, and final model assessment [S2, S3, unofficial].
*   **When not to use:** During real-time inference where speed is critical and metrics are not needed; or when stochastic perturbations are required for specific robustness tests (though this is rare for evaluation) [S2, unofficial].

## Background
NeRF Evaluation and Metric Calculation originated with the original NeRF paper [S1, S2, S3, unofficial]. It has since been extended to support various neural rendering architectures (e.g., TensoRF [S5, unofficial], 3DGS [S4, unofficial]) and diverse evaluation metrics (PSNR, SSIM, LPIPS [S3, S4, unofficial], Depth [S5, unofficial], Semantic/LiDAR [S6]). It is a standard component in almost all NeRF-based research pipelines [S1, S2, S3, S4, S5, unofficial][S6].

## Sources
[S1] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S3] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S4] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
[S5] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S6] PlaNeRF: SVD Unsupervised 3D Plane Regularization for NeRF Large-Scale Urban Scene Reconstruction, https://doi.org/10.1109/3dv62453.2024.00114, fetched 2026-11-04
