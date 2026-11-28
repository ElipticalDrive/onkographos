# Training Ray Marching and Integration

Also called: NeRF Forward Pass, NeRF Forward Pass with Ray Marching and Accumulation, Panoptic-Radiance Field Rendering, Standard NeRF forward, Training Ray Marching

## Overview
Training Ray Marching and Integration is the differentiable forward pass that synthesizes a 2D image from a 3D neural radiance field during the optimization loop [S1, unofficial][S3][S5, S10, unofficial]. It solves the problem of computing gradients for network parameters by approximating the continuous volume rendering integral as a discrete weighted sum, where weights are determined by the probability of a ray terminating at each sample point [S1, unofficial][S3][S5, S10, unofficial]. It sets itself apart from [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by enabling stochastic perturbations and adaptive sampling to facilitate gradient-based learning, rather than producing deterministic results for metric computation [S2, unofficial][S3][S5, S10, unofficial].

## How it works
### Inputs and Outputs
*   **Inputs**:
    *   `rays_o`: Ray origins, float tensor of shape $[N, 3]$ [S2, S10, unofficial].
    *   `rays_d`: Ray directions, float tensor of shape $[N, 3]$ [S2, S10, unofficial].
    *   `nears`: Near plane distances, float tensor of shape $[N]$ [S2, S10, unofficial].
    *   `fars`: Far plane distances, float tensor of shape $[N]$ [S2, S10, unofficial].
    *   `bg_color`: Background color, float tensor of shape $[3]$ [S10, unofficial].
    *   `perturb`: Boolean flag to enable noise addition to sample positions [S2, S10, unofficial].
    *   `bound`: Bounding box size scalar [S2, S10, unofficial].
    *   `density_bitfield`: Packed density bitfield, uint8 array of shape $[C \times H^3 / 8]$ [S2, S10, unofficial].
    *   `max_steps`: Maximum number of sampled points along each ray, int [S2, S10, unofficial].
    *   `dt_gamma`: Time delta gamma parameter for adaptive step sizing, float [S2, S10, unofficial].
    *   `mean_count`: Estimated mean steps to accelerate training, int32 [S2, S10, unofficial].
    *   `chunk_size`: Number of rays to process in a single forward pass to manage memory, int (default $64 \times 1024$) [S6, unofficial].
*   **Outputs**:
    *   `image`: Rendered RGB image, float tensor of shape $[N, 3]$ or $[H, W, 3]$ [S10, unofficial].
    *   `depth`: Rendered depth image, float tensor of shape $[N]$ or $[H, W]$ [S10, unofficial].
    *   `weights_sum`: Accumulated transmittance/opacity map, float tensor [S10, unofficial].

### Data Structures
*   **xyzs**: Generated point coordinates, float tensor of shape $[M, 3]$ [S2, S10, unofficial].
*   **dirs**: View directions, float tensor of shape $[M, 3]$ [S2, S10, unofficial].
*   **deltas**: Step sizes, float tensor of shape $[M, 2]$, where the first element is $\delta_t$ (segment length) and the second is $rays_t$ (ray index) [S2, unofficial].
*   **rays**: Ray metadata, int32 tensor of shape $[N, 3]$, storing `(index, point_offset, point_count)` for each ray [S2, S10, unofficial].

### Steps
1.  **Ray Generation**: Compute ray origins ($o_{world}$) and directions ($d_{world}$) in world space using camera parameters and the camera-to-world transformation [S9, unofficial].
2.  **Ray Marching (Sampling)**: March rays through the density field to generate sample points. This step uses an occupancy grid or density bitfield to prune empty space and adaptive step sizes [S2, S9, S10, unofficial]. If `perturb` is true, add uniform noise in range $[-0.5, 0.5] \times \text{sample\_dist}$ to z-values, where $\text{sample\_dist} = (\text{fars} - \text{nears}) / \text{num\_steps}$ [S10, unofficial]. Calculate near and far clipping planes for each ray using the axis-aligned bounding box (AABB) and a minimum near distance [S10, unofficial]. Generate initial uniform z-values (depths) along the ray segment between near and far [S10, unofficial].
3.  **Network Query**: Query the neural network for densities (`sigmas`) and colors (`rgbs`) at the sampled points `xyzs` and directions `dirs` [S10, unofficial]. Scale the queried densities by `density_scale` [S10, unofficial]. If `mean_count` > 0 and `force_all_rays` is false, trim `xyzs`, `dirs`, `deltas` to the actual count `m` from the step counter [S2, S10, unofficial].
4.  **Alpha Compositing (Integration)**: Integrate the sampled densities and colors along each ray to produce the final pixel color and depth [S3][S5, S10, unofficial]. Compute the distance between adjacent samples along each ray (`deltas`). For the last sample, use a large constant ($10^{10}$) to ensure the final segment contributes fully if opaque [S5, S10, unofficial]. Compute opacity, transmittance, and weights using the formulas below. Compute final RGB, depth, and accumulated transmittance. Add background color contribution and normalize depth [S10, unofficial].

### Formulas
*   Opacity: $\alpha = 1 - \exp(-\sigma \cdot d)$, where $\sigma$ is volume density and $d$ is the segment length [S5, unofficial].
*   Transmittance: $T_i = \prod_{j=0}^{i-1} \exp(-\sigma_j \delta_j)$, with $T_0 = 1$ [S3].
*   Weights: $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ [S3].
*   Final RGB color: $C = \sum w_i c_i$ [S5, unofficial].
*   Final depth: $D = \sum w_i d_i$ [S5, unofficial].
*   Accumulated transmittance: $A = \sum w_i$ [S5, unofficial].
*   Background composite: $C_{final} = C + (1 - A) \times \text{bg\_color}$ [S10, unofficial].
*   Depth normalization: $D_{final} = \max(0, D - \text{nears}) / (\text{fars} - \text{nears})$ [S10, unofficial].

### Parameters
*   `chunk_size`: $64 \times 1024$ [S6, unofficial].
*   `max_steps`: 1024 [S10, unofficial].
*   `dt_gamma`: 0 (or cone_angle in instant-ngp) [S2, S10, unofficial].
*   `near_plane`: 0.2 for MIPNeRF360, 0.0 for NeRF Synthetic [S11, unofficial].
*   `render_step_size`: $10^{-3}$ for MIPNeRF360, $5 \times 10^{-3}$ for NeRF Synthetic [S11, unofficial].
*   `alpha_thre`: $10^{-2}$ for MIPNeRF360, 0.0 for NeRF Synthetic [S11, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: Add a small epsilon ($10^{-10}$) to $1-\alpha$ in the cumulative product for transmittance to avoid numerical instability [S5, unofficial].
*   **Boundaries**: Use a large constant ($10^{10}$) for the last segment's delta to ensure it contributes fully if the volume is opaque [S5, S10, unofficial].
*   **Memory**: Process rays in chunks to avoid out-of-memory errors [S6, unofficial].
*   **Empty Space**: Use occupancy grids or density bitfields to skip empty space and accelerate training [S2, S9, S10, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def training_ray_marching_and_integration(
    rays_o: torch.Tensor, rays_d: torch.Tensor,
    nears: torch.Tensor, fars: torch.Tensor,
    bg_color: torch.Tensor, perturb: bool = True,
    max_steps: int = 1024, network_query_fn=None
):
    # 1. Generate uniform z-values between near and far planes
    num_steps = max_steps
    z_vals = torch.linspace(0.0, 1.0, num_steps, device=rays_o.device)
    z_vals = nears[..., None] * (1.0 - z_vals) + fars[..., None] * z_vals
    
    # 2. Perturb sample positions
    if perturb:
        sample_dist = (fars - nears) / num_steps
        noise = torch.rand(z_vals.shape, device=rays_o.device) * sample_dist
        z_vals = z_vals + noise
        
    # 3. Compute 3D query points
    xyzs = rays_o[..., None, :] + rays_d[..., None, :] * z_vals[..., :, None]
    
    # 4. Query neural network for density (sigma) and color (rgb)
    # In practice, this is batched or chunked to manage memory [S6, unofficial].
    sigmas, rgbs = network_query_fn(xyzs, rays_d[..., None, :].expand_as(xyzs))
    
    # 5. Compute segment lengths (deltas)
    dists = z_vals[..., 1:] - z_vals[..., :-1]
    one_e_10 = torch.tensor([1e10], dtype=dists.dtype, device=dists.device)
    dists = torch.cat([dists, one_e_10.expand(dists[..., :1].shape)], dim=-1)
    
    # 6. Compute opacity alpha
    alpha = 1.0 - torch.exp(-sigmas * dists)
    
    # 7. Compute transmittance T using cumulative product
    # T_i = product_{j=0}^{i-1} (1 - alpha_j)
    transmittance = torch.cumprod(
        torch.cat([torch.ones_like(alpha[..., :1]), 1.0 - alpha + 1e-10], dim=-1), 
        dim=-1
    )[..., :-1]
    
    # 8. Compute weights
    weights = transmittance * alpha
    
    # 9. Accumulate RGB and depth
    image = torch.sum(weights[..., None] * rgbs, dim=-2)
    depth = torch.sum(weights * z_vals, dim=-1)
    weights_sum = torch.sum(weights, dim=-1)
    
    # 10. Add background color and normalize depth
    image = image + (1.0 - weights_sum[..., None]) * bg_color
    depth = torch.clamp(depth - nears, min=0.0) / (fars - nears)
    
    return image, depth, weights_sum
```

## Subtopics

### [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md)
Renders a single image from a trained radiance field using a specified camera pose and computes quantitative fidelity metrics (PSNR, SSIM, LPIPS) against ground truth references. It differs from training ray marching by disabling stochastic perturbations to ensure deterministic, reproducible results and by operating on the full validation set rather than random minibatches. This procedure serves as the primary benchmark for comparing model variants and guiding optimization loops.

### [NeRF Network Query with Hashed Embeddings](3107-nerf-network-query-with-hashed-embeddings.md)
Queries a neural radiance field by mapping 3D spatial coordinates and 2D view directions into high-dimensional feature vectors using a multiresolution hash encoding, which are then processed by concatenated Multi-Layer Perceptrons (MLPs) to predict volume density and view-dependent color. It sets itself apart from standard NeRF [Mildenhall et al. 2020] by replacing expensive positional encoding with a learnable, sparse hash grid that enables significantly faster training convergence and real-time rendering performance on consumer hardware.

### [Chunked Network Evaluation](3177-chunked-network-evaluation.md)
A memory-management technique that evaluates a NeRF network on batched points in chunks to avoid out-of-memory errors during training or rendering. It partitions the total set of query points into smaller subsets processed sequentially, rather than attempting to process all points in a single forward pass.

### [Differentiable Volume Rendering Pipeline](5583-differentiable-volume-rendering-pipeline.md)
Orchestrates the training loop for Neural Radiance Fields by generating rays, sampling points through an implicit volume, evaluating the neural network, and aggregating results via differentiable alpha compositing. It solves the problem of optimizing 3D scene representations from 2D images by computing gradients of the rendered image loss with respect to network parameters. It differs from standard volume rendering by explicitly maintaining the computational graph through the sampling and integration steps to enable end-to-end backpropagation [G].

### [Active Learning Strategy for NeRF Distillation](7421-active-learning-strategy-for-nerf-distillation.md)
Bridges the performance gap between teacher and student Neural Radiance Field (NeRF) models during knowledge distillation by actively selecting challenging camera poses, sample rays, and sample points rather than processing all data uniformly. It distinguishes itself from standard uniform distillation by dynamically prioritizing difficult samples based on loss metrics, enabling student models to outperform those trained from scratch.

### [Direct Matching Alignment Loss](13275-direct-matching-alignment-loss.md)
A regularization term used during NeRF training to align a reference image with a set of surrogate views, thereby reducing overfitting and improving pose estimation accuracy. Unlike standard rendering losses that only compare rendered pixels against ground truth, this method enforces geometric consistency between different camera viewpoints by projecting 3D points from a reference view into surrogate views and comparing their appearance.

### [Volume Rendering Integration and Forward Pass](15462-volume-rendering-integration-and-forward-pass.md)
Defines the forward pass that synthesizes a 2D image pixel color from a 3D implicit scene representation by integrating radiance along camera rays [G]. It solves the problem of mapping a continuous 5D implicit function to a 2D image by discretizing the ray integral [G]. It sets itself apart from sibling methods like [NeRF Evaluation and Metric Calculation](3079-nerf-evaluation-and-metric-calculation.md) by focusing strictly on the geometric integration and compositing logic rather than loss computation or metric evaluation [G].

### [Lightning-NeRF Rendering and Training Loop](17111-lightning-nerf-rendering-and-training-loop.md)
A neural rendering method that accelerates training and inference by combining a hybrid density representation (voxel grid with hash encoding) with a decomposed view-independent/view-dependent MLP architecture. It distinguishes itself from standard NeRF implementations by employing adaptive sampling warmup schedules and dynamic occupancy grid updates to prune empty space, enabling efficient training on large-scale real-world scenes.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of rays and samples per ray, $O(N \times M)$, where $N$ is the number of rays and $M$ is the number of samples [S2, S10, unofficial].
*   **Memory Complexity**: Linear with respect to the number of rays and samples, $O(N \times M)$, for storing intermediate tensors [S6, unofficial].
*   **Performance**: Can be accelerated by using CUDA kernels for ray marching and compositing [S2, S10, unofficial]. Occupancy grid pruning can significantly reduce the number of samples needed, improving training speed [S9, S11, unofficial].
*   **Strengths**: Differentiable, allowing for end-to-end optimization of the neural network [S1, unofficial][S3]. Efficient with occupancy grids [S9, S11, unofficial].
*   **Limitations**: Requires careful tuning of sampling parameters [S2, S10, unofficial].
*   **When to Use**: During the training phase of NeRF models to compute gradients [S1, unofficial][S3].
*   **When Not to Use**: During evaluation or inference when deterministic results are required [S2, unofficial][S3].

## Background
Training Ray Marching and Integration is based on the volume rendering integral introduced by Mildenhall et al. in 2020 [S3][S5, unofficial]. It has been widely adopted in various NeRF implementations, including instant-ngp [S2, S10, unofficial], jaxngp [S9, unofficial], and others [S1, unofficial][S3][S5, S6, S7, S8, S11, unofficial]. The method has evolved to include optimizations such as occupancy grid pruning [S9, S11, unofficial], hash encoding [S2, S10, unofficial], and adaptive sampling [S2, S10, unofficial].

## Sources
[S1] kakaobrain/nerf-factory, https://github.com/kakaobrain/nerf-factory, fetched 2026-10-26 (unofficial)
[S2] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S3] Panoptic Neural Fields: A Semantic Object-Aware Neural Scene Representation, https://doi.org/10.1109/cvpr52688.2022.01253, fetched 2026-10-26
[S4] Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/Shyam-pi/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S5] albertgassol1/vf-nerf, https://github.com/albertgassol1/vf-nerf, fetched 2026-10-26 (unofficial)
[S6] rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, https://github.com/rigved-sanku/NeRF-Implementation-for-Efficient-3D-Scene-Reconstruction, fetched 2026-10-27 (unofficial)
[S7] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S8] city-super/GSDF, https://github.com/city-super/GSDF, fetched 2026-10-27 (unofficial)
[S9] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
[S10] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S11] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S12] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S13] Yosshi999/faster-occgrid, https://github.com/Yosshi999/faster-occgrid, fetched 2026-10-24 (unofficial)
[S14] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
[S15] FastMESH: Fast Surface Reconstruction by Hexagonal Mesh-based Neural Rendering, https://doi.org/10.48550/arxiv.2305.17858, fetched 2026-10-26 (unofficial)
[S16] LAENeRF: Local Appearance Editing for Neural Radiance Fields, https://doi.org/10.1109/cvpr52733.2024.00475, fetched 2026-10-26
[S17] Neural Radiance Fields-Based 3D Reconstruction of Power Transmission Lines Using Progressive Motion Sequence Images., https://doi.org/10.3390/s23239537, fetched 2026-10-26
[S18] DynaVol: Unsupervised Learning for Dynamic Scenes through Object-Centric Voxelization, https://doi.org/10.48550/arxiv.2305.00393, fetched 2026-10-26 (unofficial)
