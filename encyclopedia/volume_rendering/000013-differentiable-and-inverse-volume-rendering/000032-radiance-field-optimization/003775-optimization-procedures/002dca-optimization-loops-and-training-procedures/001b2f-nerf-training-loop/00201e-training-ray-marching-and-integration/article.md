# Training Ray Marching and Integration

Also called: CUDA Ray Marching (Training), NeRF Forward Pass, Standard NeRF forward, Training Ray Marching

## Overview
Training Ray Marching and Integration is the differentiable forward pass that synthesizes RGB images from a neural radiance field for gradient-based optimization [S1, unofficial][S2, unofficial][S4, unofficial]. It solves the problem of efficiently sampling 3D space along camera rays, querying a neural network for density and color, and integrating these values via alpha compositing to produce a loss-computable image [S7, unofficial][S8, unofficial]. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by computing gradients for backpropagation and from [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by including the adaptive sampling and occupancy grid pruning steps specific to the training loop [S1, unofficial][S2, unofficial][S8, unofficial].

## How it works
The method composes an implicit volume function, a point sampling scheme, and a volume renderer into a single forward pass [S2, unofficial]. It takes a bundle of rays defined by camera parameters and outputs rendered RGB images, depth maps, and accumulated transmittance [S4, unofficial][S8, unofficial].

### Inputs and Outputs
**Inputs:**
*   `rays_o`: Ray origins, float tensor of shape $[N, 3]$ [S1, unofficial][S8, unofficial].
*   `rays_d`: Ray directions, float tensor of shape $[N, 3]$ [S1, unofficial][S8, unofficial].
*   `nears`: Near plane distances, float tensor of shape $[N]$ [S1, unofficial][S8, unofficial].
*   `fars`: Far plane distances, float tensor of shape $[N]$ [S1, unofficial][S8, unofficial].
*   `density_bitfield`: Packed density bitfield, uint8 tensor of shape $[C \cdot H^3 / 8]$ [S1, unofficial][S8, unofficial].
*   `bound`: Bounding box size scalar [S1, unofficial][S8, unofficial].
*   `cascade`: Cascade parameters for multi-resolution grid [S8, unofficial].
*   `grid_size`: Grid size parameter [S8, unofficial].
*   `mean_count`: Estimated mean steps to accelerate training, int32 [S1, unofficial][S8, unofficial].
*   `perturb`: Boolean flag for adding noise to step sizes [S1, unofficial][S8, unofficial].
*   `dt_gamma`: Time delta gamma parameter for adaptive step sizing [S1, unofficial][S8, unofficial].
*   `max_steps`: Maximum number of sampled points along each ray, int [S1, unofficial][S8, unofficial].
*   `T_thresh`: Transmittance threshold for early termination [S8, unofficial].

**Outputs:**
*   `xyzs`: Generated point coordinates, float tensor of shape $[M, 3]$ [S1, unofficial][S8, unofficial].
*   `dirs`: View directions, float tensor of shape $[M, 3]$ [S1, unofficial][S8, unofficial].
*   `deltas`: Step sizes, float tensor of shape $[M, 2]$ (first is delta_t, second is rays_t) [S1, unofficial].
*   `rays`: Ray metadata, int32 tensor of shape $[N, 3]$ storing (index, point_offset, point_count) for each ray [S1, unofficial][S8, unofficial].
*   `image`: Rendered RGB image [S8, unofficial].
*   `depth`: Rendered depth image [S8, unofficial].
*   `weights_sum`: Accumulated opacity/transmittance map [S8, unofficial].

### Data Structures
*   **Occupancy Grid**: A sparse grid used to prune empty space. If `grid_prune` is enabled, it is initialized with `roi_aabb` (scene bounding box), `resolution` (e.g., 128 or 256), and `contraction_type` (e.g., AABB or UN_BOUNDED_SPHERE) [S5, unofficial].
*   **Step Counter**: An array of size 16 used to track valid samples, with `local_step` modulo 64 selecting the active slot [S6, unofficial][S8, unofficial].

### Numbered Steps

1.  **Initialization**: Ensure inputs are contiguous CUDA tensors [S1, unofficial]. Initialize the step counter array `counter` of size 16, zero it out, and increment `local_step` modulo 16 to select the active counter slot [S8, unofficial]. If `grid_prune` is true, create or update the `OccupancyGrid` using an evaluation function that approximates opacity as `density[...,None] * render_step_size` [S5, unofficial].

2.  **Ray Marching (Sampling)**: Call the CUDA kernel `raymarching.march_rays_train` with `rays_o`, `rays_d`, `bound`, `density_bitfield`, `cascade`, `grid_size`, `nears`, `fars`, `counter`, `mean_count`, `perturb`, `dt_gamma`, and `max_steps` [S8, unofficial]. This kernel generates `xyzs`, `dirs`, `deltas`, and `rays` [S8, unofficial].
    *   If `perturb` is true, generate random noise uniformly in $[0, 1]$ to jitter sample positions [S7, unofficial].
    *   If `mean_count > 0` and not `force_all_rays`, limit the output points $M$ to `mean_count` aligned to `align` [S1, unofficial].
    *   If `force_all_rays` or `mean_count <= 0`, trim `xyzs`, `dirs`, `deltas` to the actual count $m$ from `step_counter[0]` [S1, unofficial][S8, unofficial].

3.  **Neural Network Query**: Query the neural network for densities `sigmas` and colors `rgbs` at the sampled points `xyzs` and directions `dirs` [S8, unofficial]. Scale the queried densities by `density_scale` [S8, unofficial].
    *   *Chunked Evaluation*: To avoid out-of-memory errors, flatten points and view directions, iterate over chunks of size `netchunk`, call the model on each chunk, and concatenate results [S9, unofficial].

4.  **Alpha Compositing (Integration)**: Compute the distance between adjacent samples along each ray. For the last sample, use a large constant ($1e10$) to ensure the final segment contributes fully if opaque [S4, unofficial][S8, unofficial].
    *   Compute opacity $\alpha$ for each segment:
        $$ \alpha = 1 - \exp(-\sigma \cdot d) $$
        where $\sigma$ is volume density and $d$ is the distance between samples [S4, unofficial].
    *   Compute transmittance $T$:
        $$ T_i = \prod_{j=0}^{i-1} \exp(-\sigma_j \delta_j) $$
        Implemented as cumulative product: `T = torch.cumprod(torch.cat((torch.ones_like(rays_density[:1,...]), torch.exp(-rays_density * deltas)), dim=0), dim=0)` [S2, unofficial].
    *   Compute weight $w_i$ for each sample:
        $$ w_i = T_i (1 - \exp(-\sigma_i \delta_i)) $$
        Implemented as: `weights = T[:-1,...]*(1 - torch.exp(-rays_density * deltas))` [S2, unofficial].
    *   Add a small epsilon ($1e-10$) to $1-\alpha$ for numerical stability [S4, unofficial].

5.  **Aggregation**:
    *   Compute final RGB color:
        $$ C = \sum w_i c_i $$
        Implemented as: `rgb_map = sum(weights[..., None] * rbg_field, dim=-2)` [S4, unofficial].
    *   Compute final depth:
        $$ D = \sum w_i d_i $$
        Implemented as: `depth_map = sum(weights * depth_values, dim=-1)` [S4, unofficial].
    *   Compute accumulated transmittance/opacity:
        $$ A = \sum w_i $$
        Implemented as: `acc_map = weights.sum(-1)` [S4, unofficial].
    *   Add background color contribution: `comp_rgb = comp_rgb + background_color * (1.0 - opacity)` [S5, unofficial].
    *   If upsampling is enabled, compute bin weights from initial alphas and sample new z-values using stratified sampling from the PDF derived from weights [S8, unofficial]. Concatenate initial and upsampled z-values, sort them, and reorder corresponding xyzs and density outputs [S8, unofficial].

6.  **Chunk Aggregation**: If processing in chunks, concatenate outputs from all chunks along the batch dimension [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def cumprod_exclusive(tensor):
    """Mimic tf.math.cumprod(..., exclusive=True)"""
    dim = -1
    cump = torch.cumprod(tensor, dim=dim)
    cump[..., 0] = 1.0
    return cump

def render_volume_density(rbg_field, density_field, ray_origins, depth_values):
    """
    Differentiable volume rendering.
    rbg_field: (width, height, num_samples, 3)
    density_field: (width, height, num_samples)
    ray_origins: (width, height, 3)
    depth_values: (num_samples,)
    """
    one_e_10 = torch.tensor([1e10], dtype=ray_origins.dtype, device=ray_origins.device)
    dists = torch.cat((depth_values[1:] - depth_values[:-1], 
                       one_e_10.expand(depth_values[:1].shape)), dim=-1)
    dists = dists.unsqueeze(0).expand(ray_origins.shape[:-1] + (-1,))
    
    alpha = 1. - torch.exp(-density_field * dists)
    weights = alpha * cumprod_exclusive(1. - alpha + 1e-10)
    
    rgb_map = (weights[..., None] * rbg_field).sum(dim=-2)
    depth_map = (weights * depth_values).sum(dim=-1)
    acc_map = weights.sum(-1)
    
    return rgb_map, depth_map, acc_map

def training_ray_marching_and_integration(rays_o, rays_d, nears, fars, 
                                          density_bitfield, bound, cascade, grid_size,
                                          mean_count, perturb, dt_gamma, max_steps,
                                          model, chunk_size=1024*8):
    """
    Full training forward pass.
    """
    # 1. Ray Marching (CUDA Kernel)
    # CUDA kernel call: xyzs, dirs, deltas, rays = raymarching.march_rays_train(...)
    # For pseudocode completeness, we assume these are generated.
    N = rays_o.shape[0]
    M = N * max_steps
    xyzs = torch.rand(M, 3) * bound
    dirs = torch.rand(M, 3)
    deltas = torch.rand(M, 2)
    rays = torch.zeros(N, 3, dtype=torch.int32)
    rays[:, 0] = torch.arange(N)
    rays[:, 1] = torch.arange(0, M, N)
    rays[:, 2] = max_steps
    
    # 2. Neural Network Query (Chunked)
    all_rgb = []
    all_sigma = []
    for i in range(0, xyzs.shape[0], chunk_size):
        chunk_pts = xyzs[i:i+chunk_size]
        chunk_dirs = dirs[i:i+chunk_size]
        rgb_chunk, sigma_chunk = model(chunk_pts, chunk_dirs)
        all_rgb.append(rgb_chunk)
        all_sigma.append(sigma_chunk)
        
    rgbs = torch.cat(all_rgb, dim=0)
    sigmas = torch.cat(all_sigma, dim=0)
    
    # 3. Alpha Compositing
    dists = deltas[:, 0]
    alpha = 1. - torch.exp(-sigmas * dists)
    alpha_safe = alpha + 1e-10
    transmittance_prev = cumprod_exclusive(1. - alpha_safe)
    weights = transmittance_prev * alpha
    
    # 4. Aggregation per ray
    N_rays = rays_o.shape[0]
    rgb_accum = torch.zeros(N_rays, 3, device=rgbs.device)
    depth_accum = torch.zeros(N_rays, device=rgbs.device)
    opacity_accum = torch.zeros(N_rays, device=rgbs.device)
    
    ray_indices = rays[:, 0]
    ray_offsets = rays[:, 1]
    ray_counts = rays[:, 2]
    
    for r in range(N_rays):
        idx = ray_indices[r]
        offset = ray_offsets[r]
        count = ray_counts[r]
        if count > 0:
            w = weights[offset:offset+count]
            c = rgbs[offset:offset+count]
            sample_depth = torch.norm(xyzs[offset:offset+count] - rays_o[idx], dim=1)
            
            rgb_accum[idx] += torch.sum(w[:, None] * c, dim=0)
            depth_accum[idx] += torch.sum(w * sample_depth, dim=0)
            opacity_accum[idx] += torch.sum(w, dim=0)
            
    bg_color = torch.tensor([1.0, 1.0, 1.0], device=rgbs.device)
    final_rgb = rgb_accum + bg_color * (1.0 - opacity_accum[:, None])
    
    return final_rgb, depth_accum, opacity_accum
```

### Parameters and Typical Values
*   `chunk_size`: Number of rays to process in a single forward pass to manage memory. Typical values: $1024 \times 8$ [S4, unofficial][S9, unofficial].
*   `near_plane`: Near clipping plane. Default 0.2 for MIPNeRF360, 0.0 for NeRF Synthetic [S10, unofficial].
*   `render_step_size`: Step size for ray marching. Default $1e-3$ for MIPNeRF360, $5e-3$ for NeRF Synthetic [S10, unofficial].
*   `cone_angle`: Cone angle for rendering. Default 0.004 for MIPNeRF360, 0.0 for NeRF Synthetic [S10, unofficial].
*   `alpha_thre`: Alpha threshold for stopping ray marching. Default $1e-2$ for MIPNeRF360, 0.0 for NeRF Synthetic [S10, unofficial].
*   `density_scale`: Multiplier applied to network output densities [S8, unofficial].
*   `lr`: Learning rate for Adam optimizer [S2, unofficial].
*   `batch_size`: Number of random pixels sampled per iteration [S2, unofficial][S11, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: Add epsilon ($1e-10$) to $1-\alpha$ in transmittance calculation to prevent numerical instability [S4, unofficial].
*   **Boundaries**: For the last sample along a ray, use a large constant ($1e10$) for delta to ensure full contribution if the ray is opaque [S4, unofficial][S8, unofficial].
*   **Memory**: Use chunked network evaluation to avoid out-of-memory errors during training or rendering [S9, unofficial].
*   **Empty Space**: Use occupancy grid pruning to skip empty space, optimizing performance [S10, unofficial].
*   **Underestimated Mean Count**: If `mean_count` is underestimated, rays may be randomly dropped; use `force_all_rays` to disable this optimization if necessary [S1, unofficial].

## Subtopics

### [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md)
NeRF Evaluation and Metric Calculation renders a single image from a given camera pose using trained coarse and fine NeRF models, returning RGB, depth, and disparity maps. It differs from the parent method by operating in inference mode (no gradient computation for backpropagation) and by computing quantitative quality metrics (PSNR, SSIM, LPIPS) against ground truth images. It sets itself apart from sibling [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) by handling the full pipeline of ray generation, coordinate transformation (including NDC), batched rendering, and metric aggregation.

### [NeRF Network Query with Hashed Embeddings](3107-nerf-network-query-with-hashed-embeddings.md)
This method performs the neural network evaluation step within the NeRF training loop, mapping 3D spatial coordinates and viewing directions to density and color features using embedding functions and a network function. It replaces expensive high-dimensional coordinate embeddings with a structured lookup process that applies position and direction embeddings before batching the concatenated features through a network function.

### [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md)
Generates RGB images and depth maps from neural network predictions by performing volume rendering along camera rays using alpha compositing. It integrates the predicted volume density and color of sampled points into a final pixel value, accounting for occlusion via transmittance. Unlike sibling methods that focus on evaluation metrics or network query structures, this method specifically handles the mathematical integration of radiance and opacity into observable quantities.

### [NeRF-style Volume Rendering Integrator](5964-nerf-style-volume-rendering-integrator.md)
A simplified NeRF-style integrator that accumulates emission along a ray through a medium, backed by a dense grid instead of a neural network, with no direction-dependent emission. It differs from standard NeRF implementations by using a dense grid for density/emission storage and explicitly handling primal/dual modes for gradient computation in inverse rendering contexts.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of rays and samples per ray, $O(N \cdot M)$, where $N$ is the number of rays and $M$ is the number of samples [S1, unofficial][S8, unofficial]. Occupancy grid pruning reduces the effective $M$ for empty regions [S10, unofficial].
*   **Memory Complexity**: Linear with respect to the number of samples and features stored per sample [S9, unofficial]. Chunked evaluation reduces peak memory usage [S9, unofficial].
*   **Measured Performance**: Benchmarks on NeRF-synthetic dataset show PSNR scores around 33.09 at 33.7k steps using JAX/CUDA implementation [S7, unofficial].
*   **Strengths**: Differentiable, allowing gradient-based optimization of scene parameters from image supervision [S2, unofficial]. Efficient via occupancy grid pruning and adaptive step sizing [S10, unofficial].
*   **Limitations**: Memory intensive for high-resolution scenes without chunking [S9, unofficial].
*   **When to Use**: Training NeRFs and other neural radiance fields where gradient computation is required [S2, unofficial][S4, unofficial].
*   **When Not to Use**: Inference-only scenarios where speed is critical and gradients are not needed; use [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) or [NeRF Rendering (Alpha Compositing)](5124-nerf-rendering-alpha-compositing.md) instead [S1, unofficial][S2, unofficial].

## Background
Originating from the NeRF paper (Mildenhall et al. 2020), this method integrates neural implicit functions with volume rendering [S2, unofficial]. It has been adapted and optimized in various implementations such as ER-NeRF [S1, unofficial], Instant-NSR-PL [S5, unofficial], and JAXNGP [S7, unofficial] to improve training speed and memory efficiency. It is widely used in inverse rendering and 3D scene reconstruction tasks [S2, unofficial][S11, unofficial].

## Sources
[S1] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S2] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S3] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S4] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S5] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S6] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S7] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S8] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S9] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S10] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S11] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
