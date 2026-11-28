# NeRF Evaluation and Metric Calculation

Also called: Evaluation and Metric Calculation, NeRF Evaluation, NeRF Validation, Single Image Rendering, Validation Rendering

## Overview
NeRF Evaluation and Metric Calculation renders a single image from a trained radiance field using a specified camera pose and computes quantitative fidelity metrics against ground truth references [S1, S2, S3, unofficial]. It differs from training ray marching by disabling stochastic perturbations to ensure deterministic, reproducible results [S2, unofficial]. This procedure serves as the primary benchmark for comparing model variants and guiding optimization loops [S2, S3, unofficial].

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
- `metrics`: Dictionary containing PSNR and SSIM values [S2, S3, unofficial].

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
   - For each chunk, call the rendering function with `perturb=False` [S1, S2, unofficial].
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
8. **Output**: Return the rendered images and computed metrics [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np
from skimage.metrics import structural_similarity as compare_ssim

def evaluate_nerf(coarse_model, fine_model, pose, K, H, W, near, far, args, device, gt_image=None):
    # 1. Initialization
    coarse_model.eval()
    fine_model.eval()
    with torch.no_grad():
        # 2. Input Conversion
        if isinstance(pose, np.ndarray):
            pose = torch.from_numpy(pose).float().to(device)
        if isinstance(K, np.ndarray):
            K = torch.from_numpy(K).float().to(device)

        # 3. Ray Generation
        rays_o, rays_d = get_rays(H, W, K, pose)

        # 4. NDC Conversion
        if args.use_ndc:
            focal = K[0, 0].item()
            rays_o, rays_d = ndc_rays(H, W, focal, 1.0, rays_o, rays_d)

        # 5. Flatten rays
        rays_o = rays_o.reshape(-1, 3)
        rays_d = rays_d.reshape(-1, 3)

        # 6. Chunked Rendering
        chunk = args.chunk
        all_rgb = []
        all_depth = []
        all_disp = []

        for i in range(0, rays_o.shape[0], chunk):
            chunk_o = rays_o[i:i+chunk]
            chunk_d = rays_d[i:i+chunk]

            # Step the sources do not give: full volume integration loop is implemented in [Volume Rendering Integration and Forward Pass](15462-volume-rendering-integration-and-forward-pass.md)
            result = render_rays(
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

            rgb_key = 'rgb_fine' if 'rgb_fine' in result else 'rgb_coarse'
            depth_key = 'depth_fine' if 'depth_fine' in result else 'depth_coarse'
            disp_key = 'disp_fine' if 'disp_fine' in result else 'disp_coarse'

            all_rgb.append(result[rgb_key].reshape(-1, 3).cpu().numpy())
            all_depth.append(result[depth_key].reshape(-1).cpu().numpy())
            all_disp.append(result[disp_key].reshape(-1).cpu().numpy())

        # 7. Result Aggregation
        rgb_image = np.concatenate(all_rgb, axis=0).reshape(H, W, 3)
        depth_image = np.concatenate(all_depth, axis=0).reshape(H, W)
        disp_image = np.concatenate(all_disp, axis=0).reshape(H, W)

        # 8. Metric Calculation
        metrics = {}
        if gt_image is not None:
            gt_rgb = gt_image.reshape(H, W, 3)
            mse = np.mean((rgb_image - gt_rgb) ** 2)
            psnr = -10.0 * np.log10(mse) if mse > 0 else float('inf')
            metrics['psnr'] = psnr
            ssim = compare_ssim(rgb_image, gt_rgb, full=True, multichannel=True)
            metrics['ssim'] = ssim

        return rgb_image, depth_image, disp_image, metrics

def get_rays(H, W, K, c2w):
    i, j = np.meshgrid(np.arange(W, dtype=np.float32),
                       np.arange(H, dtype=np.float32), indexing='xy')
    dirs = np.stack([(i - K[0, 2].item()) / K[0, 0].item(),
                     -(j - K[1, 2].item()) / K[1, 1].item(),
                     -np.ones_like(i)], axis=-1)
    dirs = dirs.reshape(-1, 3)
    rays_d = np.dot(dirs, c2w[:3, :3].T.numpy())
    rays_o = np.broadcast_to(c2w[:3, 3].numpy(), rays_d.shape).copy()
    return rays_o, rays_d

def ndc_rays(H, W, focal, near, rays_o, rays_d):
    rays_o = rays_o.reshape(-1, 3)
    rays_d = rays_d.reshape(-1, 3)
    rays_o = rays_o - rays_d * near
    ndc_scale = 2.0 * focal / W
    rays_o[:, :2] = rays_o[:, :2] / ndc_scale
    rays_d[:, :2] = rays_d[:, :2] / ndc_scale
    rays_o[:, 2] = rays_o[:, 2] + near
    return rays_o, rays_d

def render_rays(rays_o, rays_d, coarse_model, fine_model, near, far,
                n_samples_coarse, n_samples_fine, perturb, raw_noise_std,
                white_bkgd, lindisp, netchunk):
    # Step the sources do not give: full volume integration loop is implemented in [Volume Rendering Integration and Forward Pass](15462-volume-rendering-integration-and-forward-pass.md)
    pass
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

### [MIMO Objective](10931-mimo-objective.md)
The MIMO objective is the total training loss for MIMO-NeRF, combining pixel-wise reconstruction errors with a 3D consistency regularization term across $M$ reformulated multi-input multi-output (MIMO) neural radiance fields. It enforces consistency between parallel MLPs trained with different sampling configurations, enabling faster rendering without sacrificing quality. Choose it when training multiple parallel NeRFs to accelerate inference.

### [Novel View Selection Policy](12895-novel-view-selection-policy.md)
This policy selects the most informative synthetic views for active learning in neural radiance fields by quantifying information gain through uncertainty metrics. It distinguishes itself from heuristic baselines (Random, FVS) by using epistemic uncertainty as a proxy for information gain, consistently outperforming them in reconstruction quality. Choose it when actively selecting new camera poses for training.

### [Standard Metric Computation](13741-standard-metric-computation.md)
Standard Metric Computation quantifies the fidelity of rendered images against ground truth references to evaluate the quality of novel-view synthesis in Neural Radiance Fields (NeRF). It addresses the problem of assessing rendering quality by balancing pixel-level precision, structural integrity, and perceptual similarity, providing a standardized benchmark for comparing different NeRF architectures and training procedures. Choose it for standard benchmarking of novel view synthesis.

### [Depth Supervision and Pseudo-Depth](13742-depth-supervision-and-pseudo-depth.md)
This family of techniques incorporates geometric constraints into the radiance field optimization loop to improve reconstruction quality, particularly in low-data regimes or when explicit geometry is required for downstream tasks. While standard NeRF optimization relies solely on photometric RGB losses, these methods introduce depth-related penalties or sampling strategies. Depth Supervision explicitly minimizes the difference between rendered depth and ground-truth depth (derived from SfM keypoints or stereo), whereas Pseudo-Depth approximates surface geometry via weighted averaging of sampled points to guide feature distillation or segmentation without requiring explicit depth labels. Choose it when geometric accuracy is prioritized over photometric fidelity.

### [Ray-Based Volumetric Uncertainty Estimator](17852-ray-based-volumetric-uncertainty-estimator.md)
The Ray-Based Volumetric Uncertainty Estimator computes the uncertainty of the underlying 3D geometry in a Neural Radiance Field (NeRF) by calculating the entropy of the weight distribution of color samples along each ray. It treats the opacity function as a cumulative distribution function (CDF) and its derivative as a probability density function (PDF), allowing the weight distribution to serve as a proxy for geometric certainty. Unlike explicit 3D representations or RGB-D fusion methods, this approach leverages the implicit neural shape representation to estimate information gain without relying on structure-from-motion or depth sensors. It is distinct from methods like S-NeRF or NeRF-W as it requires no model modification or additional training, functioning as a post-hoc analysis plug-in. Choose it for uncertainty-aware scene analysis without retraining.

### [Depth-Regularizing Loss Functions](17876-depth-regularizing-loss-functions.md)
Depth-Regularizing Loss Functions constrain the geometric consistency of differentiable volume rendering by penalizing discrepancies between rendered depth maps and external geometric estimates, such as stereo-matching outputs or optical flow correspondences. Unlike photometric losses that only enforce color fidelity, these methods explicitly regularize the scene's implicit geometry, accelerating convergence and improving robustness in textureless or occluded regions. They are distinguished from sibling methods like [MIMO Objective](10931-mimo-objective.md) (which focuses on multi-scale image features) and [Ray-Based Volumetric Uncertainty Estimator](17852-ray-based-volumetric-uncertainty-estimator.md) (which models aleatoric uncertainty) by their direct reliance on explicit depth or flow supervision signals. Choose them when training stability in textureless regions is required.

### [Active Scout Tracking with NeRF](20507-active-scout-tracking-with-nerf.md)
Active Scout Tracking with NeRF is a multi-target tracking method for pursuit-evasion games in highly occluded urban environments, where a quadrotor scout tracks dynamic targets on the ground using Neural Radiance Fields (NeRFs) built online from RGB and depth images. It solves the problem of selecting the next best view for a scout to maximize information gain about target locations and unknown parts of the city. Unlike [Standard Metric Computation](13741-standard-metric-computation.md) which evaluates static performance, this method actively drives the scout's trajectory by maximizing mutual information between future observations and past observations, conditioned on the future scout location. It differs from [MIMO Objective](10931-mimo-objective.md) by focusing on single-scout active perception rather than multi-agent coordination, and from [Novel View Selection Policy](12895-novel-view-selection-policy.md) by explicitly integrating NeRF-based scene representation with Bayes filter target tracking. Choose it for active robotic perception and tracking.

### Expected Termination Depth Synthesis
Synthesizes depth maps from a trained NeRF's density field by computing the expected termination depth of a ray [S4]. It casts a ray towards the camera and uses the density field to determine how deep into the volumetric object the ray is expected to penetrate, using that distance as the depth map value [S4]. NeRF does not directly estimate object boundaries nor directly produce depth maps; hard surfaces are simulated using a very dense but not infinitely dense field [S4]. It differs from direct depth sensors (e.g., RealSense D415) which fail on thin/reflective objects, and from MVS (e.g., COLMAP) which assumes Lambertian surfaces and fails on specularities [S4].

### Ray Integration-based Depth Estimation
A method to convert the continuous density field of a trained NeRF into a discrete surface point cloud for DSM generation by computing a weighted average of sampled point depths [S5]. It computes the final depth $D_f$ for each ray using the weighted sum of sample depths, calculates 3D coordinates $P$ using the ray origin $o$, direction $d$, and computed depth $D_f$, transforms point cloud coordinates into geographic latitude and longitude using the RPC model, projects the coordinates into a 2D planar coordinate system through Universal Transverse Mercator (UTM) projection, and maps elevation values to grid cells based on predefined spatial resolution and grid range to complete DSM generation [S5]. The formula is $D_f = \sum_{i=1}^{N} T_i \alpha_i z_i$, where $z_i$ is the depth of the $i$-th sample, $\alpha_i$ is the opacity, and $T_i$ is the transmittance [S5].

### Visual Hull Density Masking
Computes visual hulls from ground-truth masks of training views to remove extraneous geometry blobs from reconstructions [S6]. During training pose reconstruction, it computes visual hulls from ground-truth masks of training views and sets density values outside the visual hulls to 0 [S6]. During testing on unseen poses, it disables the mask usage [S6]. Without masking, extraneous geometry blobs appear in reconstructions by Neural Body and Ani-NeRF [S6].

### Camera Pose Perturbation Evaluation
Evaluates reconstruction stability under camera parameter uncertainty by running Monte Carlo simulations [S7]. It defines perturbation ranges for translation ($\Delta T$) and rotation ($\Delta R$), initializes an iteration counter to 0, and while iterations $< N$: randomly samples $\Delta T \in [-T_{\max}, T_{\max}]$ and $\Delta R \in [-R_{\max}, R_{\max}]$, applies perturbations to camera parameters, runs the reconstruction pipeline, computes RMSE between generated reconstruction and reference data, stores RMSE, and increments the counter [S7]. It computes Average RMSE = mean(stored RMSEs), Error Range = max(stored RMSEs) - min(stored RMSEs), and Uncertainty = Error Range / 2 [S7]. The formula is $\text{Uncertainty} = (\text{Max RMSE} - \text{Min RMSE}) / 2$ [S7]. Typical values: $N=30$, $R_{\max}=2^\circ$ (Scenario A) or $4^\circ$ (Scenario B) [S7]. On the Ignatius dataset, Scenario A ($\pm 20$ mm, $\pm 2^\circ$) yields Average RMSE = 19.72 mm, Uncertainty = 2.95 mm [S7].

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
[S6] ARAH: Animatable Volume Rendering of Articulated Human SDFs, https://doi.org/10.1007/978-3-031-19824-3_1, fetched 2026-10-26
[S7] A Critical Analysis of NeRF-Based 3D Reconstruction, https://doi.org/10.3390/rs15143585, fetched 2026-10-26
