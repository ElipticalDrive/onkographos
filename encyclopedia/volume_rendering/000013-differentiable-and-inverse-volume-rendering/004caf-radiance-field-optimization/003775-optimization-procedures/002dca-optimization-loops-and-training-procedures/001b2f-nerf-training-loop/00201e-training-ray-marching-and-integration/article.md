# Training Ray Marching and Integration

Also called: CUDA Ray Marching (Training), NeRF Forward Pass, Standard NeRF forward, Training Ray Marching, VolumeRenderer

## Overview
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images and depth maps from a Neural Radiance Field (NeRF) during optimization [S1, S2, S4, unofficial]. It solves the problem of computing gradients for the neural network parameters by sampling points along camera rays, querying volume density and color, and integrating them via alpha compositing [S4, S9, unofficial]. It sets itself apart from evaluation rendering by using stochastic sampling with perturbation and adaptive step sizes to facilitate gradient-based optimization, whereas evaluation uses deterministic rendering for reproducible metrics [S2, S9, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origins $\mathbf{o}$ and directions $\mathbf{d}$ (float $[N, 3]$), near/far planes (float $[N]$ or scalars), bounding box size `bound` (float), density bitfield (uint8 $[C \cdot H^3 / 8]$), cascade parameters (float), grid size (int), maximum steps `max_steps` (int), perturbation flag (bool), and optional mean count `mean_count` (int32) [S1, S9, unofficial].
*   **Outputs:** Sampled 3D coordinates $\mathbf{xyzs}$ (float $[M, 3]$), view directions $\mathbf{dirs}$ (float $[M, 3]$), step sizes $\mathbf{deltas}$ (float $[M, 2]$ where first is $\delta_t$ and second is ray time $t$), and ray metadata $\mathbf{rays}$ (int32 $[N, 3]$ storing index, point offset, and point count) [S1, S9, unofficial].

**Data Structures:**
*   **Density Bitfield:** A packed uint8 array of shape $[C \cdot H^3 / 8]$ representing a coarse occupancy grid used for pruning empty space [S1, S9, unofficial].
*   **Ray Metadata:** An integer array of shape $[N, 3]$ where each row contains the ray index, the starting offset in the flattened output arrays, and the number of valid samples for that ray [S1, S9, unofficial].

**Numbered Steps:**

1.  **Ray Generation:** Compute ray origins and directions in world space using camera intrinsics (focal length, principal point) and extrinsics (camera-to-world transform) [S8, S4, unofficial].
2.  **Bound Calculation:** Calculate near ($t_{start}$) and far ($t_{end}$) clipping planes for each ray by intersecting with the scene's axis-aligned bounding box (AABB) and applying a minimum near distance [S8, S9, unofficial].
3.  **Initial Sampling:** Generate initial uniform depth values (z-values) along the ray segment between near and far, scaled by the number of steps [S9, unofficial].
4.  **Perturbation:** If `perturb` is true, add uniform noise in range $[-0.5, 0.5] \cdot \text{sample\_dist}$ to the z-values, where $\text{sample\_dist} = (\text{fars} - \text{nears}) / \text{num\_steps}$ [S8, S9, unofficial].
5.  **Ray Marching (CUDA):** Call the backend CUDA kernel `march_rays_train` with ray origins, directions, bounds, density bitfield, cascade, grid size, near/far planes, counter, mean count, perturbation flag, block size (128), force all rays flag, time delta gamma `dt_gamma`, and max steps [S9, unofficial]. This kernel uses the occupancy grid to skip empty space and returns sampled points $\mathbf{xyzs}$, directions $\mathbf{dirs}$, deltas $\mathbf{deltas}$, and ray indices $\mathbf{rays}$ [S7, S9, unofficial].
6.  **Trimming:** If `force_all_rays` is false and `mean_count` > 0, trim the output arrays to the actual count $m$ recorded in `step_counter[0]` [S1, S9, unofficial]. If `mean_count` <= 0, process all rays [S1, S9, unofficial].
7.  **Neural Query:** Query the neural network for volume density $\sigma$ (sigmas) and RGB color at the sampled points $\mathbf{xyzs}$ and directions $\mathbf{dirs}$ [S9, unofficial]. Scale densities by `density_scale` if applicable [S9, unofficial].
8.  **Delta Calculation:** Compute the distance between adjacent samples along each ray. For the last sample, append a large constant ($10^{10}$) to ensure the final segment contributes fully if opaque [S4, S2, unofficial].
9.  **Alpha Compositing:** Compute opacity $\alpha$ for each segment using $\alpha = 1 - \exp(-\sigma \cdot d)$, where $d$ is the segment length [S4, unofficial]. Compute transmittance $T_i = \prod_{j=0}^{i-1} \exp(-\sigma_j \delta_j)$ with $T_0=1$ [S2, unofficial]. Compute weights $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ [S2, unofficial].
10. **Integration:** Compute final RGB color $C = \sum w_i c_i$ and depth $D = \sum w_i d_i$ [S4, S2, unofficial]. Add background color contribution $(1 - \sum w_i) \cdot \text{bg\_color}$ [S6, S9, unofficial].

**Formulas:**
*   **Opacity:** $\alpha = 1 - \exp(-\sigma \cdot d)$ [S4, unofficial].
*   **Transmittance:** $T_i = \prod_{j=0}^{i-1} \exp(-\sigma_j \delta_j)$ [S2, unofficial].
*   **Weight:** $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ [S2, unofficial].
*   **RGB:** $C = \sum w_i c_i$ [S4, unofficial].
*   **Depth:** $D = \sum w_i d_i$ [S4, unofficial].

**Parameters:**
*   `chunk_size`: Number of rays to process in a single forward pass to manage memory (default $64 \times 1024$) [S5, unofficial].
*   `perturb`: Boolean flag for adding noise to step sizes (default True for training) [S9, unofficial].
*   `dt_gamma`: Time delta gamma parameter for adaptive step sizing (cone angle) [S9, S1, unofficial].
*   `max_steps`: Maximum number of sampled points along each ray (default 1024) [S9, unofficial].
*   `mean_count`: Estimated mean steps to accelerate training; rays are randomly dropped if actual count exceeds this [S9, S1, unofficial].
*   `T_thresh`: Transmittance threshold for early termination in compositing [S9, unofficial].
*   `density_scale`: Multiplier applied to network output densities [S9, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Add a small epsilon ($10^{-10}$) to $1-\alpha$ in the cumulative product for transmittance to prevent numerical instability [S4, S2, unofficial].
*   **Boundaries:** Use a large constant ($10^{10}$) for the last segment's delta to handle opaque surfaces correctly [S4, S2, unofficial].
*   **Memory:** Use chunked evaluation if the number of samples per ray multiplied by the number of rays exceeds GPU VRAM capacity [S1, S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def cumprod_exclusive(tensor: torch.Tensor) -> torch.Tensor:
    """Mimic tf.math.cumprod(..., exclusive=True) for PyTorch."""
    dim = -1
    cump = torch.cumprod(tensor, dim=dim)
    cump[..., 0] = 1.0
    return cump

def render_volume_density(rbg_field: torch.Tensor, density_field: torch.Tensor, ray_origins: torch.Tensor, depth_values: torch.Tensor) -> tuple:
    """
    Differentiably renders a radiance field.
    Args:
        rbg_field: Tensor of shape (width, height, num_samples, 3) for RGB.
        density_field: Tensor of shape (width, height, num_samples) for sigma.
        ray_origins: Tensor of shape (width, height, 3).
        depth_values: Tensor of shape (num_samples).
    Returns:
        rgb_map, depth_map, acc_map
    """
    one_e_10 = torch.tensor([1e10], dtype=ray_origins.dtype, device=ray_origins.device)
    # Compute distances between consecutive samples
    dists = torch.cat((depth_values[..., 1:] - depth_values[..., :-1],
                       one_e_10.expand(depth_values[..., :1].shape)), dim=-1)
    
    # Compute opacity alpha
    alpha = 1. - torch.exp(-density_field * dists)
    
    # Compute weights using transmittance
    # T_i = product of exp(-sigma_j * delta_j) for j < i
    # weights = alpha * T
    transmittance = cumprod_exclusive(1. - alpha + 1e-10)
    weights = alpha * transmittance
    
    # Aggregate RGB and Depth
    rgb_map = (weights[..., None] * rbg_field).sum(dim=-2)
    depth_map = (weights * depth_values).sum(dim=-1)
    acc_map = weights.sum(-1)
    
    return rgb_map, depth_map, acc_map

def train_ray_marching_and_integration(rays_o, rays_d, nears, fars, bound, density_bitfield, cascade, grid_size, max_steps, perturb=True, mean_count=-1, dt_gamma=0):
    """
    Simulates the training ray marching and integration loop.
    Note: The actual ray marching is done via a CUDA kernel in practice.
    This pseudocode outlines the logic and integration steps.
    """
    # 1. Ray Generation and Bounds (Simplified)
    # In practice, rays_o and rays_d are generated from camera parameters.
    # nears and fars are computed by intersecting rays with AABB.
    
    # 2. Initial Sampling
    num_samples = max_steps
    depth_values = torch.linspace(nears, fars, num_samples).to(rays_o)
    
    # 3. Perturbation
    if perturb:
        sample_dist = (fars - nears) / num_samples
        noise = torch.rand_like(depth_values) * sample_dist * 0.5 - sample_dist * 0.25 # Uniform [-0.5, 0.5] * sample_dist
        depth_values = depth_values + noise
        
    # 4. Ray Marching (CUDA Kernel Simulation)
    # In reality, this calls: xyzs, dirs, deltas, rays = raymarching.march_rays_train(...)
    # For pseudocode, we assume we have sampled points along the rays
    xyzs = rays_o[..., None, :] + rays_d[..., None, :] * depth_values[..., :, None]
    dirs = rays_d[..., None, :].expand_as(xyzs)
    
    # 5. Neural Query (Simulated)
    # In reality, this queries the MLP: sigmas, rgbs = model(xyzs, dirs)
    # Here we simulate random density and color for demonstration
    sigmas = torch.rand_like(xyzs[..., 0]) * 0.1 # Placeholder for network output
    rgbs = torch.rand_like(xyzs) # Placeholder for network output
    
    # 6. Delta Calculation
    # Compute dists between consecutive samples
    one_e_10 = torch.tensor([1e10], dtype=xyzs.dtype, device=xyzs.device)
    dists = torch.cat((depth_values[..., 1:] - depth_values[..., :-1],
                       one_e_10.expand(depth_values[..., :1].shape)), dim=-1)
    
    # 7. Alpha Compositing
    alpha = 1. - torch.exp(-sigmas * dists)
    transmittance = cumprod_exclusive(1. - alpha + 1e-10)
    weights = alpha * transmittance
    
    # 8. Integration
    rgb_map = (weights[..., None] * rgbs).sum(dim=-2)
    depth_map = (weights * depth_values).sum(dim=-1)
    acc_map = weights.sum(-1)
    
    # 9. Background Color
    # Add background color for rays that do not accumulate full opacity
    bg_color = torch.tensor([1.0, 1.0, 1.0], dtype=rgb_map.dtype, device=rgb_map.device)
    rgb_map = rgb_map + bg_color * (1.0 - acc_map[..., None])
    
    return rgb_map, depth_map, acc_map
```

## Subtopics

### CUDA Ray Marching (Training), NeRF Forward Pass, Standard NeRF forward, Training Ray Marching, Training Ray Marching and Integration — NeRF Forward Pass, march_rays_train, run_cuda
A rendering function that uses an occupancy grid estimator to skip empty space during volume rendering, optimizing performance [S10, unofficial]. It sets itself apart by using a dense grid for density/emission storage and explicitly handling primal/dual modes for gradient computation in inverse rendering contexts [S10, unofficial]. It is chosen when performance is critical and the scene has large empty regions [S10, unofficial]. Parameters include `near_plane` (default 0.2 for MIPNeRF360, 0.0 for NeRF Synthetic), `render_step_size` (default 1e-3 for MIPNeRF360, 5e-3 for NeRF Synthetic), `render_bkgd` (background color tensor), `cone_angle` (default 0.004 for MIPNeRF360, 0.0 for NeRF Synthetic), and `alpha_thre` (default 1e-2 for MIPNeRF360, 0.0 for NeRF Synthetic) [S10, unofficial].

### Differentiable Volume Rendering Pipeline
Orchestrates rendering by instantiating an implicit volume, a sampling scheme, and a renderer [S11, unofficial]. It solves the problem of integrating these components into a single forward pass for training [S11, unofficial]. It sets itself apart by looping over cameras, generating NDC grids, creating ray bundles, passing them through a sampler, and evaluating the implicit volume [S11, unofficial]. It is chosen for general NeRF training pipelines [S11, unofficial]. The loss function is $\mathcal{L} = \frac{1}{N} \sum_{i=1}^{N} (\hat{C}_i - C_i)^2$, where $\hat{C}_i$ is the rendered feature/color, $C_i$ is the ground truth color, and $N$ is the batch size [S11, unofficial]. Parameter `cfg.training.batch_size` controls memory usage and gradient noise [S11, unofficial].

### NeRF Forward Pass with Alpha Masking and Two-Stage Shading
Sets optimization flags, samples points (using NDC simulation if enabled), computes distances, and handles alpha masks to mark rays invalid if they hit empty space [S12, unofficial]. It sets itself apart by combining ray validity with alpha masks and adjusting distances by ray norm in NDC mode [S12, unofficial]. It is chosen when using alpha masks for geometry constraints [S12, unofficial]. Steps include normalizing coordinates, computing density features, converting to sigma, computing alpha/weight/background weight, and adding background color for rays with low opacity [S12, unofficial].

### Progressive Strided Ray Sampling
A progressive sampling strategy to stabilize GAN training, starting with large stride ray generation and progressively reducing stride size to cover larger regions initially and focus on local regions later [S13]. It sets itself apart by dynamically changing the sampling stride $s$ during training [S13]. It is chosen for GAN-based NeRF training [S13]. The $K \times K$ patch $P$ of stride $s$ containing point $(u, v)$ is defined as: $\mathcal{P}(u, v, s)=\left \{(u +sx, v +sy) \mid x, y \in \left \{0, \ldots , {K}\right \}\right \}$ [S13].

### Coarse-to-Fine Optimization Scheme
Initial optimization uses multi-view stereo normal maps for supervision to rapidly converge mesh geometry, then shifts to RGB and mask loss after remeshing [S14, unofficial]. It sets itself apart by using normal supervision in the early stages and remeshing at specific iterations (100, 200, 400) to recover fine details [S14, unofficial]. It is chosen for surface reconstruction tasks [S14, unofficial]. The mask loss is $L_{mask} = 1 - IOU(\hat{M}, M)$ [S14, unofficial]. Learning rate $lr = 1e-3$ using Adam, decayed at remeshing iterations, and weight $\lambda_2 = 50$ for mask and normal losses [S14, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear in the number of rays and samples per ray, $O(N \cdot M)$, where $N$ is the number of rays and $M$ is the number of samples [S1, S9, unofficial]. Adaptive marching reduces effective $M$ by pruning empty space [S1, S9, unofficial].
*   **Memory Complexity:** $O(N \cdot M)$ for storing intermediate tensors (xyzs, dirs, deltas, sigmas, rgbs) [S1, S9, unofficial]. Chunked evaluation reduces peak memory to $O(\text{chunk\_size} \cdot M)$ [S5, unofficial].
*   **Measured Performance:** Benchmarks show significant speedups with occupancy grid pruning and early termination [S8, unofficial]. For example, instant-NSR-PL achieves competitive PSNR on NeRF-Synthetic datasets with optimized ray marching [S8, unofficial].
*   **Scaling:** Scales well with GPU parallelism due to CUDA implementation [S1, S9, unofficial]. Memory usage is the primary bottleneck for high-resolution scenes [S1, S3, unofficial].
*   **Strengths:** Differentiable, efficient with occupancy grids, supports adaptive sampling [S1, S9, unofficial].
*   **Limitations:** Memory intensive for large scenes, sensitive to hyperparameters like `dt_gamma` and `max_steps` [S1, S9, unofficial].
*   **When to Use:** Standard training of NeRFs and related radiance field models [S1, S2, S4, unofficial].
*   **When Not to Use:** Evaluation where deterministic results are required (use [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md)) [S2, unofficial].

## Background
Originated from the NeRF paper by Mildenhall et al. 2020 [S2, S4, unofficial]. It has been developed and optimized in various implementations such as TalkingGaussian [S1, unofficial], instant-NSR-PL [S6, unofficial], jaxngp [S8, unofficial], and IDDR-NGP [S9, unofficial]. It is widely used in 3D scene reconstruction, novel view synthesis, and inverse rendering applications [S1, S2, S4, S8, S9, unofficial].

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S3] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S4] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S5] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S6] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S7] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S8] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S9] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S10] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S11] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S12] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S13] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
[S14] FastMESH: Fast Surface Reconstruction by Hexagonal Mesh-based Neural Rendering, https://doi.org/10.48550/arxiv.2305.17858, fetched 2026-10-26 (unofficial)
