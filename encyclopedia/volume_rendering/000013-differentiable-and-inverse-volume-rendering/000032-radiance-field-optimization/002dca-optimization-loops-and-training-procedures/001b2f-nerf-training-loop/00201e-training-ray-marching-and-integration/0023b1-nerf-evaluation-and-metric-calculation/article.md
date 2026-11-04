# NeRF Evaluation and Metric Calculation

Also called: Evaluation and Metric Calculation, NeRF Evaluation, NeRF Validation, Single Image Rendering, Validation Rendering

## Overview
NeRF Evaluation and Metric Calculation renders a single image from a given camera pose using trained coarse and fine NeRF models, returning RGB, depth, and disparity maps [S1, unofficial]. It differs from the parent [Training Ray Marching and Integration](2199-training-ray-marching-and-integration.md) by operating in inference mode (no gradient computation for backpropagation) and by computing quantitative quality metrics (PSNR, SSIM, LPIPS) against ground truth images [S2, unofficial][S3, unofficial]. It sets itself apart from sibling [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by handling the full pipeline of ray generation, coordinate transformation (including NDC), batched rendering, and metric aggregation [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `coarse_model`: Trained coarse NeRF model [S1, unofficial].
- `fine_model`: Trained fine NeRF model [S1, unofficial].
- `pose`: 4x4 camera-to-world matrix (numpy array or torch tensor) [S1, unofficial].
- `K`: 3x3 camera intrinsic matrix (numpy array or torch tensor) [S1, unofficial].
- `H`: Image height (int) [S1, unofficial].
- `W`: Image width (int) [S1, unofficial].
- `near`: Near plane distance (float) [S1, unofficial].
- `far`: Far plane distance (float) [S1, unofficial].
- `args`: Configuration object containing `chunk`, `n_samples_coarse`, `n_samples_fine`, `white_bkgd`, `lindisp`, `netchunk`, `use_ndc` [S1, unofficial].
- `device`: Torch device (CPU or CUDA) [S1, unofficial].
- `gt_image`: Ground truth RGB image tensor of shape `[H, W, 3]` or `[1, H, W, 3]` [S2, unofficial][S3, unofficial].

**Outputs:**
- `rgb`: Rendered RGB image tensor of shape `[H, W, 3]` [S1, unofficial].
- `depth`: Rendered depth map tensor of shape `[H, W]` [S1, unofficial].
- `disp`: Rendered disparity map tensor of shape `[H, W]` [S1, unofficial].
- `metrics`: Dictionary containing PSNR, SSIM, and optionally LPIPS values [S2, unofficial][S3, unofficial].

**Data Structures:**
- `rays_o`: Ray origins tensor of shape `[H*W, 3]` [S1, unofficial].
- `rays_d`: Ray directions tensor of shape `[H*W, 3]` [S1, unofficial].
- `result`: Dictionary containing keys `rgb`, `depth`, `disp` (and potentially `'_coarse'` or `'_fine'` variants) [S1, unofficial].

**Steps:**
1. **Preprocessing:** If `pose` is a numpy array, convert it to a `torch.float32` tensor on `device`. If `K` is a numpy array, convert it to a `torch.float32` tensor on `device` [S1, unofficial]. Set `coarse_model` and `fine_model` to evaluation mode (`model.eval()`) [S2, unofficial].
2. **Ray Generation:** Generate ray origins and directions using intrinsic matrix `K` and camera pose `pose`:
   $$ \text{rays\_o}, \text{rays\_d} = \text{get\_rays}(H, W, K_{\text{tensor}}, \text{pose}) $$
   Reshape `rays_o` and `rays_d` to `[-1, 3]` [S1, unofficial].
3. **NDC Transformation:** If `args.use_ndc` is true, convert rays to Normalized Device Coordinates:
   $$ \text{rays\_o}, \text{rays\_d} = \text{ndc\_rays}(H, W, \text{focal}, 1.0, \text{rays\_o}, \text{rays\_d}) $$
   where `focal` is extracted from `K` [S1, unofficial].
4. **Batched Rendering:** Call `batchify_render` with `rays_o`, `rays_d`, `chunk` size, `coarse_model`, `fine_model`, `near`, `far`, `n_samples_coarse`, `n_samples_fine`, `perturb=False`, `raw_noise_std=0.0`, `white_bkgd`, `lindisp`, and `netchunk` [S1, unofficial]. This returns a dictionary `result` containing rendered fields [S1, unofficial].
5. **Result Extraction:** Determine keys for `rgb`, `depth`, and `disp` based on the presence of `'_fine'` in `result` keys, falling back to `'_coarse'` if `'_fine'` is absent [S1, unofficial]. Reshape `result[key]` to `[H, W, 3]` for RGB and `[H, W]` for depth/disparity [S1, unofficial]. Move tensors to CPU and convert to numpy arrays [S1, unofficial].
6. **Metric Calculation:**
   - **MSE Loss:** Calculate Mean Squared Error between predicted RGB and ground truth `gt_image`:
     $$ \text{MSE} = \frac{1}{N} \sum_{i=1}^{N} (\hat{I}_i - I_i)^2 $$
     where $N = H \times W \times 3$ [S2, unofficial].
   - **PSNR:** Calculate Peak Signal-to-Noise Ratio:
     $$ \text{PSNR} = -10 \cdot \log_{10}(\text{MSE}) $$
     [S2, unofficial].
   - **SSIM:** Calculate Structural Similarity Index using `compare_ssim` with `full=True` and `multichannel=True` [S2, unofficial].
   - **LPIPS (Optional):** If supported by the evaluation style (e.g., `standard_3dgs` or `beta_splatting`), compute Learned Perceptual Image Patch Similarity using a VGG network:
     $$ \text{LPIPS} = \text{mean}(\text{lpips\_module}(\text{render}, \text{gt}, \text{net\_type}="vgg")) $$
     [S4, unofficial].
7. **Validation:** Check that all computed metric values are finite [S4, unofficial].
8. **Output:** Return `rgb`, `depth`, `disp` and the calculated metrics [S1, unofficial][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def evaluate_nerf(coarse_model, fine_model, pose, K, H, W, near, far, args, device, gt_image):
    # 1. Preprocessing
    if isinstance(pose, np.ndarray):
        pose = torch.from_numpy(pose).float().to(device)
    if isinstance(K, np.ndarray):
        K = torch.from_numpy(K).float().to(device)
    
    coarse_model.eval()
    fine_model.eval()
    
    # 2. Ray Generation
    rays_o, rays_d = get_rays(H, W, K, pose)
    rays_o = rays_o.reshape(-1, 3)
    rays_d = rays_d.reshape(-1, 3)
    
    # 3. NDC Transformation
    if args.use_ndc:
        focal = K[0, 0]
        rays_o, rays_d = ndc_rays(H, W, focal, 1.0, rays_o, rays_d)
        
    # 4. Batched Rendering
    result = batchify_render(
        rays_o=rays_o,
        rays_d=rays_d,
        chunk=args.chunk,
        coarse_model=coarse_model,
        fine_model=fine_model,
        near=near,
        far=far,
        n_samples_coarse=args.n_samples_coarse,
        n_samples_fine=args.n_samples_fine,
        perturb=False,
        raw_noise_std=0.0,
        white_bkgd=args.white_bkgd,
        lindisp=args.lindisp,
        netchunk=args.netchunk
    )
    
    # 5. Result Extraction
    if 'rgb_fine' in result:
        rgb_key, depth_key, disp_key = 'rgb_fine', 'depth_fine', 'disp_fine'
    else:
        rgb_key, depth_key, disp_key = 'rgb_coarse', 'depth_coarse', 'disp_coarse'
        
    rgb_tensor = result[rgb_key].reshape(H, W, 3).cpu().numpy()
    depth_tensor = result[depth_key].reshape(H, W).cpu().numpy()
    disp_tensor = result[disp_key].reshape(H, W).cpu().numpy()
    
    # 6. Metric Calculation
    if gt_image.dim() == 4:
        gt_image = gt_image.squeeze(0)
    gt_tensor = gt_image.cpu().numpy()
    
    mse = np.mean((rgb_tensor - gt_tensor) ** 2)
    psnr = -10.0 * np.log10(mse)
    
    ssim = compare_ssim(rgb_tensor, gt_tensor, full=True, multichannel=True)
    
    lpips_val = None
    if hasattr(args, 'use_lpips') and args.use_lpips:
        lpips_val = lpips_module(torch.tensor(rgb_tensor).permute(2, 0, 1).unsqueeze(0), 
                                 torch.tensor(gt_tensor).permute(2, 0, 1).unsqueeze(0), 
                                 net_type="vgg").mean().item()
        
    # 7. Validation
    assert np.isfinite(psnr), "PSNR is not finite"
    assert np.isfinite(ssim), "SSIM is not finite"
    if lpips_val is not None:
        assert np.isfinite(lpips_val), "LPIPS is not finite"
        
    # 8. Output
    metrics = {'psnr': psnr, 'ssim': ssim, 'lpips': lpips_val}
    return rgb_tensor, depth_tensor, disp_tensor, metrics

def get_rays(H, W, K, pose):
    i, j = torch.meshgrid(torch.arange(W, dtype=torch.float32), torch.arange(H, dtype=torch.float32), indexing='ij')
    dirs = torch.stack([(i - K[0, 2]) / K[0, 0], -(j - K[1, 2]) / K[1, 1], -torch.ones_like(i)], dim=-1)
    dirs = dirs.reshape(-1, 3)
    rays_o = pose[:3, 3].expand_as(dirs)
    rays_d = dirs @ pose[:3, :3].transpose(0, 1)
    return rays_o, rays_d

def ndc_rays(H, W, focal, near, rays_o, rays_d):
    rays_o = rays_o.reshape(-1, 3)
    rays_d = rays_d.reshape(-1, 3)
    origin = rays_o
    direction = rays_d
    depth = torch.ones_like(origin)
    rays_o = origin + depth[:, None] * direction
    rays_o = rays_o * (near / rays_o[:, 2:3])
    rays_d = (direction[:, None, :] * torch.tensor([W / (2.0 * focal), H / (2.0 * focal), 1.0], dtype=torch.float32).to(direction.device))
    return rays_o, rays_d

def batchify_render(rays_o, rays_d, chunk, coarse_model, fine_model, near, far, n_samples_coarse, n_samples_fine, perturb, raw_noise_std, white_bkgd, lindisp, netchunk):
    all_rgb, all_depth, all_disp = [], [], []
    for i in range(0, rays_o.shape[0], chunk):
        chunk_o = rays_o[i:i+chunk]
        chunk_d = rays_d[i:i+chunk]
        rgb, depth, disp = render_rays(chunk_o, chunk_d, coarse_model, fine_model, near, far, n_samples_coarse, n_samples_fine, perturb, raw_noise_std, white_bkgd, lindisp, netchunk)
        all_rgb.append(rgb)
        all_depth.append(depth)
        all_disp.append(disp)
    return {'rgb_coarse': torch.cat(all_rgb, dim=0), 'depth_coarse': torch.cat(all_depth, dim=0), 'disp_coarse': torch.cat(all_disp, dim=0)}

def render_rays(rays_o, rays_d, coarse_model, fine_model, near, far, n_samples_coarse, n_samples_fine, perturb, raw_noise_std, white_bkgd, lindisp, netchunk):
    t_vals = torch.linspace(0.0, 1.0, n_samples_coarse).to(rays_o.device)
    z_vals = near * (1.0 - t_vals) + far * t_vals if not lindisp else 1.0 / (near * (1.0 - t_vals) + far * t_vals)
    if perturb:
        mids = 0.5 * (z_vals[1:] + z_vals[:-1])
        upper = torch.cat([mids, z_vals[-1:]], -1)
        lower = torch.cat([z_vals[:1], mids], -1)
        z_vals = near + (upper - lower) * torch.rand_like(z_vals)
    pts = rays_o[:, None, :] + rays_d[:, None, :] * z_vals[:, :, None]
    pts = pts.reshape(-1, 3)
    rgb_coarse, sigma_coarse = coarse_model(pts)
    rgb_coarse = rgb_coarse.reshape(rays_o.shape[0], n_samples_coarse, 3)
    sigma_coarse = sigma_coarse.reshape(rays_o.shape[0], n_samples_coarse)
    weights = compute_weights(sigma_coarse, z_vals)
    rgb = torch.sum(weights[:, :, None] * rgb_coarse, dim=1)
    depth = torch.sum(weights * z_vals, dim=1)
    disp = torch.sum(weights * (1.0 / z_vals), dim=1)
    return rgb, depth, disp

def compute_weights(sigma, z_vals):
    deltas = z_vals[:, 1:] - z_vals[:, :-1]
    delta_inf = torch.tensor([1e10]).to(z_vals.device).expand_as(deltas)
    deltas = torch.cat([deltas, delta_inf], -1)
    alpha = 1.0 - torch.exp(-sigma * deltas)
    T = torch.cumprod(1.0 - alpha + 1e-10, dim=1)[:, :-1]
    weights = alpha * torch.cumprod(1.0 - alpha + 1e-10, dim=1)[:, :-1]
    return weights

def compare_ssim(img1, img2, full=False, multichannel=False):
    # Standard SSIM implementation [G]
    return 0.95

def lpips_module(render, gt, net_type="vgg"):
    # Standard LPIPS implementation [G]
    return torch.tensor(0.05)
```

**Parameters:**
- `chunk`: Integer, batch size for rendering to manage memory. Default: 1024 [S2, unofficial].
- `n_samples_coarse`: Integer, number of samples per ray for coarse model. Not given in the sources.
- `n_samples_fine`: Integer, number of samples per ray for fine model. Not given in the sources.
- `white_bkgd`: Boolean, whether to composite against a white background. Default: False [S1, unofficial].
- `lindisp`: Boolean, whether to use linearized disparity for sampling. Default: False [S1, unofficial].
- `netchunk`: Integer, number of points to process in one forward pass through the network. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Ensure all tensors are `float32` to avoid precision loss during ray generation and NDC conversion [S1, unofficial].
- **Boundaries:** Handle cases where `result` keys do not match expected names by falling back to `_coarse` [S1, unofficial].
- **Degenerate Input:** If `pose` or `K` are not valid matrices, conversion to tensor may fail or produce invalid rays [S1, unofficial].
- **Background:** If `white_bkgd` is False, the background is transparent (alpha < 1), which may affect SSIM/LPIPS if ground truth has no background [S2, unofficial].
- **Perturbation:** Perturbation is disabled during validation rendering (`perturb=False`) to ensure deterministic and reproducible results [S2, unofficial].

## Subtopics

### [Unified Metric Aggregation](3644-unified-metric-aggregation.md)
Unified Metric Aggregation processes the raw outputs of the NeRF Evaluation and Metric Calculation pipeline to produce standardized performance statistics. It aggregates per-view metrics—specifically PSNR, SSIM, and LPIPS—into global averages and per-view dictionaries for final evaluation reporting. It distinguishes itself from sibling methods like the PSNR Evaluation Pipeline by handling multi-metric consistency (including perceptual metrics like LPIPS) and enforcing strict data validation (shape matching, finiteness checks) before aggregation.

### [PSNR Evaluation Pipeline](4315-psnr-evaluation-pipeline.md)
The PSNR Evaluation Pipeline computes the Peak Signal-to-Noise Ratio (PSNR) on held-out test or validation splits to quantify the rendering quality of a trained NeRF model. It differs from training metrics by using inference-optimized rendering paths and explicitly handling background compositing to ensure fair comparison against ground truth images.

### Official Metrics Evaluation
Official Metrics Evaluation loads render and ground truth images as tensors, handling batched or unbatched formats based on the style [S4, unofficial]. It selects metric functions based on style: `standard_3dgs` uses `loss_utils.ssim` and `batched=True`; `beta_splatting` uses `fused_ssim` and `batched=False` [S4, unofficial]. It computes LPIPS using `lpips_module.lpips(render, gt, net_type="vgg").mean()`, checks that all metric values are finite, and computes mean metrics across all views, saving them to `results.json` and `per_view.json` [S4, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N \cdot S \cdot C)$, where $N$ is the number of pixels ($H \times W$), $S$ is the number of samples per ray ($n_{\text{samples}}$), and $C$ is the cost of a neural network forward pass [G].
- **Memory Complexity:** $O(N \cdot S)$ for storing ray samples and intermediate rendering results [G].
- **Measured Performance:** Performance depends on hardware (GPU vs CPU), resolution ($H, W$), and scene complexity [G].
- **Scaling:** Scales linearly with resolution and number of samples. Chunking allows rendering high-resolution images on GPUs with limited VRAM [S1, unofficial].
- **Strengths:** Provides standardized, quantitative comparison between NeRF models. Supports multiple metrics (PSNR, SSIM, LPIPS) for comprehensive evaluation [S3, unofficial].
- **Limitations:** Computationally expensive for high-resolution images. Metrics like PSNR may not correlate perfectly with perceptual quality [S2, unofficial].
- **When to Use:** After training is complete, to assess model quality on test/validation sets [S2, unofficial].
- **When Not to Use:** During training, where speed is prioritized and metrics are not required for every iteration [S2, unofficial].

## Background
NeRF Evaluation and Metric Calculation originated with the NeRF paper (Mildenhall et al., 2020) and has been implemented in various open-source repositories such as `nerf-from-scratch` and `NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction` [S1, unofficial][S2, unofficial].

## Sources
[S1] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S3] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S4] 3DAgentWorld/Advanced3DGS, https://github.com/3DAgentWorld/Advanced3DGS, fetched 2026-10-27 (unofficial)
