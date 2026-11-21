# NeRF Evaluation and Metric Calculation

Also called: Evaluation and Metric Calculation, NeRF Evaluation, NeRF Validation, Single Image Rendering, Validation Rendering

## Overview
NeRF Evaluation and Metric Calculation renders a single image from a trained radiance field using a specified camera pose and computes quantitative fidelity metrics (PSNR, SSIM, LPIPS) against ground truth references [S1, S2, S3, unofficial]. It differs from training ray marching by disabling stochastic perturbations to ensure deterministic, reproducible results and by operating on the full validation set rather than random minibatches [S2, unofficial]. This procedure serves as the primary benchmark for comparing model variants and guiding optimization loops [S2, S3, unofficial].

## How it works
**Inputs:**
- `coarse_model`: Trained coarse NeRF MLP [S1, unofficial].
- `fine_model`: Trained fine NeRF MLP [S1, unofficial].
- `pose`: 4x4 camera-to-world matrix (numpy array or torch tensor) [S1, S2, unofficial].
- `K`: 3x3 camera intrinsic matrix (numpy array or torch tensor) [S1, S2, unofficial].
- `H`, `W`: Image height and width in pixels [S1, S2, unofficial].
- `near`, `far`: Near and far plane distances along the ray [S1, S2, unofficial].
- `args`: Configuration object containing `chunk` (default 1024), `n_samples_coarse`, `n_samples_fine`, `white_bkgd`, `lindisp`, `netchunk`, `use_ndc` [S1, unofficial].
- `device`: Torch device (CPU/GPU) [S1, unofficial].

**Outputs:**
- `rgb`: Rendered RGB image of shape `[H, W, 3]` in `[0, 1]` range [S1, S2, unofficial].
- `depth`: Rendered depth map of shape `[H, W]` [S1, unofficial].
- `disp`: Rendered disparity map of shape `[H, W]` [S1, unofficial].
- `metrics`: Dictionary containing PSNR, SSIM, and optionally LPIPS values [S2, S3, unofficial].

**Data Structures:**
- Rays are represented as two 3D vectors: `rays_o` (origins) and `rays_d` (directions), flattened to `[-1, 3]` before processing [S1, unofficial].
- Results are stored in a dictionary with keys `'rgb'`, `'depth'`, `'disp'`, and potentially `'_coarse'` or `'_fine'` suffixes depending on the model configuration [S1, unofficial].

**Steps:**
1. **Initialization**: Set `coarse_model` and `fine_model` to evaluation mode (`model.eval()`) to disable dropout and batch normalization updates [S2, S3, unofficial]. Disable gradient computation using `torch.no_grad()` [S2, unofficial].
2. **Input Conversion**: If `pose` or `K` are numpy arrays, convert them to `torch.float32` tensors on the specified `device` [S1, unofficial].
3. **Ray Generation**: Compute ray origins and directions using the intrinsic matrix `K`, image dimensions `H, W`, and camera pose `pose` [S1, S3, unofficial].
   - Formula for `K` construction (if not provided):
     $$ K = \begin{bmatrix} f & 0 & 0.5 W \\ 0 & f & 0.5 H \\ 0 & 0 & 1 \end{bmatrix} $$
     where $f$ is the focal length in pixels [S1, unofficial].
4. **NDC Conversion (Optional)**: If `args.use_ndc` is true, convert ray origins and directions to Normalized Device Coordinates (NDC) using the focal length, near plane, and ray parameters [S1, unofficial].
5. **Chunked Rendering**: Process rays in chunks of size `chunk` (default 1024) to manage GPU memory [S1, S2, unofficial].
   - For each chunk, call the rendering function (`batchify_render` or `render_rays`) with `perturb=False` [S1, S2, unofficial].
   - The rendering function performs volume integration:
     $$ C(r) = \sum_{i=1}^{N} T_i \alpha_i c_i $$
     where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ is the transmittance, $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ is the opacity, $\sigma_i$ is density, $c_i$ is color, and $\delta_i$ is the distance between samples [S1, unofficial][S5].
   - Depth is computed as the expected termination depth:
     $$ D_f = \sum_{i=1}^{N} T_i \alpha_i z_i $$
     where $z_i$ is the depth of the $i$-th sample [S5].
6. **Result Aggregation**: Concatenate results from all chunks, reshape to `[H, W, 3]` for RGB and `[H, W]` for depth/disparity, and move to CPU [S1, S2, unofficial].
7. **Metric Calculation**:
   - **MSE**: Calculate Mean Squared Error between predicted RGB and ground truth RGB [S2, S3, unofficial].
   - **PSNR**: Calculate Peak Signal-to-Noise Ratio:
     $$ \text{PSNR} = -10 \cdot \log_{10}(\text{MSE}) $$
     [S3, unofficial].
   - **SSIM**: Calculate Structural Similarity Index using `compare_ssim` with `full=True` and `multichannel=True` [S3, unofficial].
   - **LPIPS**: (Optional) Calculate Learned Perceptual Image Patch Similarity [S6].
8. **Output**: Return the rendered images and computed metrics [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from skimage.metrics import structural_similarity as compare_ssim

def evaluate_nerf(coarse_model, fine_model, pose, K, H, W, near, far, args, device, gt_image=None):
    """
    Render a single image and compute metrics.
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
    rays_o, rays_d = get_rays(H, W, K, pose)
    
    # 4. NDC conversion if enabled
    if args.use_ndc:
        focal = K[0, 0]
        rays_o, rays_d = ndc_rays(H, W, focal, 1.0, rays_o, rays_d)
        
    # 5. Flatten rays
    rays_o = rays_o.reshape(-1, 3)
    rays_d = rays_d.reshape(-1, 3)
    
    # 6. Chunked rendering
    chunk = args.chunk
    all_rgb = []
    all_depth = []
    all_disp = []
    
    with torch.no_grad():
        for i in range(0, rays_o.shape[0], chunk):
            chunk_o = rays_o[i:i+chunk]
            chunk_d = rays_d[i:i+chunk]
            
            # Call rendering function (implementation details in sibling articles)
            # perturb=False is critical for deterministic evaluation
            result = batchify_render(
                chunk_o, chunk_d, 
                coarse_model, fine_model, 
                near, far, 
                args.n_samples_coarse, args.n_samples_fine,
                perturb=False,
                raw_noise_std=0.0,
                white_bkgd=args.white_bkgd,
                lindisp=args.lindisp,
                netchunk=args.netchunk
            )
            
            # Extract results based on keys
            rgb_key = 'rgb_fine' if 'rgb_fine' in result else 'rgb_coarse'
            depth_key = 'depth_fine' if 'depth_fine' in result else 'depth_coarse'
            disp_key = 'disp_fine' if 'disp_fine' in result else 'disp_coarse'
            
            rgb_chunk = result[rgb_key].reshape(-1, 3).cpu().numpy()
            depth_chunk = result[depth_key].reshape(-1).cpu().numpy()
            disp_chunk = result[disp_key].reshape(-1).cpu().numpy()
            
            all_rgb.append(rgb_chunk)
            all_depth.append(depth_chunk)
            all_disp.append(disp_chunk)
            
    # 7. Reshape to image dimensions
    rgb_image = np.concatenate(all_rgb, axis=0).reshape(H, W, 3)
    depth_image = np.concatenate(all_depth, axis=0).reshape(H, W)
    disp_image = np.concatenate(all_disp, axis=0).reshape(H, W)
    
    # 8. Compute metrics if ground truth is provided
    metrics = {}
    if gt_image is not None:
        gt_rgb = gt_image.reshape(H, W, 3)
        
        # MSE
        mse = np.mean((rgb_image - gt_rgb) ** 2)
        
        # PSNR
        if mse > 0:
            psnr = -10.0 * np.log10(mse)
        else:
            psnr = float('inf')
        metrics['psnr'] = psnr
        
        # SSIM
        ssim = compare_ssim(rgb_image, gt_rgb, full=True, multichannel=True)
        metrics['ssim'] = ssim
        
        # LPIPS (requires external library)
        # metrics['lpips'] = calculate_lpips(rgb_image, gt_rgb)
        
    return rgb_image, depth_image, disp_image, metrics

def get_rays(H, W, K, c2w):
    """
    Generate rays from camera intrinsics and pose.
    """
    i, j = np.meshgrid(np.arange(W, dtype=np.float32),
                       np.arange(H, dtype=np.float32), indexing='xy')
    dirs = np.stack([(i - K[0, 2]) / K[0, 0],
                     -(j - K[1, 2]) / K[1, 1],
                     -np.ones_like(i)], axis=-1)
    
    # Rotate ray directions from camera frame to world frame
    # c2w is 4x4, rotation is top-left 3x3
    dirs = dirs.reshape(-1, 3)
    rays_d = np.dot(dirs, c2w[:3, :3].T)
    rays_o = np.broadcast_to(c2w[:3, 3], rays_d.shape).copy()
    
    return rays_o, rays_d
```

**Parameters:**
- `chunk`: Integer, default 1024. Number of rays processed per batch to limit memory usage [S1, S2, unofficial].
- `perturb`: Boolean, default `False` for evaluation. Controls whether jitter is added to sample positions [S2, unofficial].
- `white_bkgd`: Boolean. Whether to composite the result against a white background [S1, unofficial].
- `lindisp`: Boolean. Whether to sample linearly in disparity space instead of depth space [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Perturbation**: Must be disabled (`perturb=False`) during validation to ensure deterministic and reproducible results [S2, unofficial].
- **Memory**: Large images or high sample counts may exceed GPU memory; chunking is essential [S1, S2, unofficial].
- **NDC**: If using NDC coordinates, rays must be transformed correctly before rendering [S1, unofficial].
- **Key Extraction**: The result dictionary keys may vary (`_coarse` vs `_fine`); code must handle fallbacks [S1, unofficial].

## Subtopics

### [Masked Evaluation](7531-masked-evaluation.md)
Masked Evaluation restricts metric computation to pixels visible in at least two training views, excluding occluded or single-view-only regions to prevent artificially inflated errors. It operates on rendered depth maps to determine visibility.

### [MIMO Objective](10931-mimo-objective.md)
The MIMO Objective is a training loss function, not an evaluation metric, but is listed as a sibling. It enforces consistency between parallel NeRFs to enable faster rendering.

### [Novel View Selection Policy](12895-novel-view-selection-policy.md)
This policy uses epistemic uncertainty, derived from evaluation metrics, to select informative synthetic views for active learning.

### [Text-Image Direction Score](13695-text-image-direction-score.md)
TIDS measures semantic alignment between text prompts and rendered changes using CLIP embeddings, distinct from photometric metrics.

### [Standard Metric Computation](13741-standard-metric-computation.md)
Calculates PSNR, SSIM, and LPIPS to quantify photorealistic fidelity.

### [Depth Supervision and Pseudo-Depth](13742-depth-supervision-and-pseudo-depth.md)
Techniques that incorporate geometric constraints into the optimization loop, using rendered depth for supervision.

### [Depth-Supervised EndoNeRF Loss](15276-depth-supervised-endonerf-loss.md)
A specific training objective combining photometric and depth-supervised losses to accelerate convergence.

### Expected Termination Depth Synthesis
Synthesizes depth maps by computing the expected termination depth of a ray in the density field, noting that NeRF does not directly estimate boundaries [S4].

### Ray Integration-based Depth Estimation
Computes final depth $D_f$ as a weighted sum of sample depths ($D_f = \sum T_i \alpha_i z_i$) to generate point clouds for DSMs [S5].

### Optical Flow Loss
A loss function derived from NeRF depth calculations to supervise view synthesis using optical flow correspondences between adjacent views [S6].

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of pixels and samples per ray. Chunking adds overhead but prevents OOM errors [S1, S2, unofficial].
- **Memory Complexity**: Proportional to `chunk` size. Full image rendering requires storing intermediate tensors for all rays in the chunk [S1, unofficial].
- **Performance**: Deterministic rendering is faster than training due to disabled perturbations and gradient computations [S2, unofficial].
- **Strengths**: Provides objective, quantitative comparison of model quality [S2, S3, unofficial].
- **Limitations**: Metrics like PSNR/SSIM do not always correlate with perceptual quality; LPIPS is more expensive to compute [S6].

## Background
Originated with the NeRF paper (Mildenhall et al., 2020), evaluation procedures became standard for benchmarking novel view synthesis [S1, S2, S3, unofficial]. The method has been adapted for various NeRF variants, including those using hashed embeddings, chunked evaluation, and differentiable pipelines [S1, S2, S3, unofficial].

## Sources
[S1] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S2] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S3] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S4] NeRF-Supervision: Learning Dense Object Descriptors from Neural Radiance Fields, https://arxiv.org/abs/2203.01913v2, fetched 2026-10-26
[S5] Shading and Geometric Constraint Neural Radiance Field for DSM Reconstruction from Multi-View Satellite Images, https://doi.org/10.3390/rs18071091, fetched 2026-11-05
[S6] Bidirectional Optical Flow NeRF: High Accuracy and High Quality under Fewer Views, https://doi.org/10.1609/aaai.v37i1.25109, fetched 2026-10-29
