# NeRF Volumetric Rendering (Coarse/Fine Sampling)

Also called: Vanilla NeRF, Standard NeRF, NeRF Two-Stage Sampling, Volume Rendering (NeRF), NeRF Coarse-to-Fine Rendering, Volumetric Rendering with MLP, Neural Radiance Field (NeRF) Rendering, Neural Volume Rendering (Alpha Blending)

## Overview

NeRF Volumetric Rendering (Coarse/Fine Sampling) is a differentiable rendering method that synthesizes novel views of complex 3D scenes by training a continuous volumetric radiance field represented by a multi-layer perceptron (MLP) [S3][S9, unofficial]. It solves the problem of aliasing and noise in neural volume rendering by employing a two-stage sampling strategy: a coarse network samples uniformly (or with stratified sampling) to estimate the scene's density distribution, and a fine network samples more densely in high-density regions identified by the coarse stage to produce a refined color estimate [S15, S24, unofficial]. This method sets itself apart from its siblings, such as [NeRF Importance Sampling](1565-nerf-importance-sampling.md) which refines sampling via a probability density function derived from coarse weights, and [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md) which uses learned acceleration structures, by relying on a simple, deterministic two-stage MLP query with uniform/stratified coarse sampling and denser fine sampling without auxiliary probability density estimation or complex data structures [S1, S15, S24, unofficial].

## How it works

**Inputs and Outputs:**
*   **Inputs:** A set of training images with known camera poses (extrinsics and intrinsics) [S3]. For rendering, a camera pose $P$ (rotation $R$ and translation $t$) and image dimensions $H \times W$ [S9, unofficial].
*   **Outputs:** Rendered RGB image, accumulated opacity (alpha), depth map, and disparity map [S1, S11, unofficial].

**Data Structures:**
*   **Neural Network ($f_\theta$):** An MLP mapping 5D coordinates (3D position $\mathbf{x}$, 3D viewing direction $\mathbf{d}$) to raw density $\hat{\sigma}$ and raw color $\hat{\mathbf{c}}$ [S4, S32].
    *   **Input:** Positional encoding $\gamma(\mathbf{x})$ and $\gamma(\mathbf{d})$ [S2, S32].
    *   **Output:** Raw density $\hat{\sigma} \in \mathbb{R}$ and raw color $\hat{\mathbf{c}} \in \mathbb{R}^3$ [S1, unofficial][S32].
*   **Ray Batch:** A tensor of shape $[R, \dots]$ containing ray origins $\mathbf{o}$, directions $\mathbf{d}$, near bounds $t_n$, and far bounds $t_f$ [S7, S10, unofficial].
*   **Sample Points:** A tensor of shape $[R, N, 3]$ representing 3D coordinates along rays [S7, unofficial].
*   **Network Outputs:** Tensors of shape $[R, N, 4]$ containing raw density and color for each sample [S1, unofficial].

**Parameters:**
*   $N_{coarse}$: Number of samples per ray for the coarse stage. Typical value: 64 [S7, S33, unofficial].
*   $N_{fine}$: Number of samples per ray for the fine stage. Typical value: 128 [S33, unofficial].
*   $t_n, t_f$: Near and far plane bounds. For bounded scenes in NDC coordinates, $t_n=0, t_f=1$ [S30, unofficial].
*   `perturb`: Boolean flag for stratified sampling jitter. Default: True [S7, S25, unofficial].
*   `lindisp`: Boolean flag for sampling in inverse depth. Default: False [S6, S9, unofficial].
*   `raw_noise_std`: Standard deviation of noise added to raw density for regularization. Default: 1.0 [S11, unofficial].
*   `chunk_size`: Number of rays processed in a single forward pass to manage memory [S13].

**Numbered Steps:**

1.  **Ray Generation:** For each pixel $(u, v)$ in the target image, compute the ray origin $\mathbf{o}$ and direction $\mathbf{d}$ from the camera pose $P$ and intrinsics [S4, S32].
    *   $\mathbf{o} = t$
    *   $\mathbf{d} = R K^{-1} [u, v, 1]^T$ (normalized or unnormalized depending on implementation, typically unnormalized in original NeRF code to preserve metric depth scaling) [S9, unofficial][S32].

2.  **Coarse Sampling:** Generate $N_{coarse}$ sample depths $t_i$ along the ray between $t_n$ and $t_f$ [S7, S9, unofficial].
    *   Compute linearly spaced values $t\_vals \in [0, 1]$ of size $N_{coarse}$ [S7, unofficial].
    *   Map to depth:
        *   If `lindisp` is False: $z\_vals = t_n (1 - t\_vals) + t_f t\_vals$ [S6, S9, unofficial].
        *   If `lindisp` is True: $z\_vals = \frac{1}{\frac{1}{t_n}(1 - t\_vals) + \frac{1}{t_f}t\_vals}$ [S6, S9, unofficial].
    *   If `perturb` is True, apply stratified sampling:
        *   Compute midpoints: $mids = 0.5 (z\_vals_{1:} + z\_vals_{:-1})$ [S7, unofficial].
        *   Define intervals: $lower = [z\_vals_{:1}, mids]$, $upper = [mids, z\_vals_{-1:}]$ [S7, unofficial].
        *   Sample uniform random noise $u \sim \text{Uniform}(0, 1)$ and update: $z\_vals = lower + (upper - lower) \cdot u$ [S7, S25, unofficial].

3.  **Coarse Network Query:** Compute 3D points $\mathbf{x}_i = \mathbf{o} + z\_vals_i \mathbf{d}$ [S7, S10, unofficial]. Query the coarse MLP $f_{\theta_{coarse}}$ at these points with viewing directions $\mathbf{d}$ to obtain raw density $\hat{\sigma}_i$ and raw color $\hat{\mathbf{c}}_i$ [S1, S7, unofficial].
    *   Apply activation functions:
        *   Density: $\sigma_i = \text{softplus}(\hat{\sigma}_i + b)$ or $\text{ReLU}(\hat{\sigma}_i)$ [S8, S11, unofficial]. Note: Original NeRF uses softplus [S8, unofficial], but many implementations use ReLU or no activation with noise [S11, unofficial]. We cite the softplus/ReLU variants as present in sources.
        *   Color: $\mathbf{c}_i = \text{sigmoid}(\hat{\mathbf{c}}_i)$ [S1, S11, unofficial].

4.  **Coarse Compositing:** Compute segment lengths $\delta_i = z\_vals_{i+1} - z\_vals_i$ for $i=0 \dots N_{coarse}-2$, and set $\delta_{N_{coarse}-1} = 10^{10}$ (or a large value) to ensure the ray terminates [S11, unofficial][S13].
    *   Compute alpha: $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ [S7, S28, unofficial].
    *   Compute transmittance: $T_i = \exp(-\sum_{j=0}^{i-1} \sigma_j \delta_j)$, with $T_0 = 1$ [S14, unofficial][S32].
    *   Compute weights: $w_i = T_i \alpha_i$ [S1, S24, unofficial].
    *   Compute coarse color: $\mathbf{C}_{coarse} = \sum_{i=0}^{N_{coarse}-1} w_i \mathbf{c}_i$ [S1, S24, unofficial].
    *   Compute coarse depth: $D_{coarse} = \sum_{i=0}^{N_{coarse}-1} w_i z\_vals_i$ [S23][S24, unofficial].
    *   Compute coarse opacity: $A_{coarse} = \sum_{i=0}^{N_{coarse}-1} w_i$ [S24, unofficial].

5.  **Fine Sampling:** Generate $N_{fine}$ additional sample depths $z\_vals_{fine}$ [S24, unofficial].
    *   The sources describe this as sampling from the probability distribution defined by the coarse weights $w_i$ (excluding the last one) [S24, S31, unofficial].
    *   *Note:* The specific algorithm for "sampling finer z-values" using coarse weights is detailed in [NeRF Importance Sampling](1565-nerf-importance-sampling.md). In the standard coarse/fine pipeline, this often involves drawing samples from the CDF of the coarse weights [S31, unofficial]. For implementation completeness, if importance sampling is not used, the fine samples are often placed uniformly or stratified within the intervals defined by the coarse samples, weighted by the coarse alpha [S24, unofficial]. However, the source [S24, unofficial] explicitly states "sample_finer_z_values implements stratified sampling from the probability distribution defined by coarse weights". We assume the standard NeRF fine sampling draws from the coarse opacity distribution [S31, unofficial].
    *   Concatenate coarse $z\_vals$ and fine $z\_vals_{fine}$, sort them, and remove duplicates or handle overlaps [S24, unofficial]. Let the total number of samples be $N_{total} = N_{coarse} + N_{fine}$ (or interleaved) [S24, unofficial].
    *   *Crucial Step:* Detach the fine sample positions from the computation graph to prevent gradients from flowing back to the coarse network during the fine pass [S24, unofficial].

6.  **Fine Network Query:** Query the fine MLP $f_{\theta_{fine}}$ at the sorted sample points $\mathbf{x}_i$ and directions $\mathbf{d}$ to obtain $\hat{\sigma}_i, \hat{\mathbf{c}}_i$ [S15, S25, unofficial].
    *   Apply activations: $\sigma_i = \text{softplus}(\hat{\sigma}_i)$, $\mathbf{c}_i = \text{sigmoid}(\hat{\mathbf{c}}_i)$ [S8, S11, unofficial].

7.  **Fine Compositing:** Recompute $\delta_i, \alpha_i, T_i, w_i$ using the sorted fine samples [S1, S24, unofficial].
    *   Compute fine color: $\mathbf{C}_{fine} = \sum_{i=0}^{N_{total}-1} w_i \mathbf{c}_i$ [S1, S24, unofficial].
    *   Compute fine depth: $D_{fine} = \sum_{i=0}^{N_{total}-1} w_i z\_vals_i$ [S23][S24, unofficial].
    *   Compute fine opacity: $A_{fine} = \sum_{i=0}^{N_{total}-1} w_i$ [S24, unofficial].

8.  **Loss Computation:**
    *   Compute photometric loss (MSE) between $\mathbf{C}_{coarse}$ and ground truth $\mathbf{C}_{gt}$ [S15, unofficial].
    *   Compute photometric loss (MSE) between $\mathbf{C}_{fine}$ and ground truth $\mathbf{C}_{gt}$ [S15, unofficial].
    *   Total Loss = $L_{coarse} + L_{fine}$ [S15, unofficial].

9.  **Optimization:** Update $\theta_{coarse}$ and $\theta_{fine}$ using gradient descent [S3, S12].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn

def positional_encoding(x, num_bands=10):
    """Standard positional encoding as defined in [S2, S19]."""
    # x is [..., D]
    # Output is [..., D * 2 * num_bands]
    freq_bands = 2.0 ** torch.linspace(0, num_bands - 1, num_bands).to(x.device)
    x_proj = x.unsqueeze(-1) * freq_bands.unsqueeze(0) # [..., D, num_bands]
    x_proj = torch.cat([torch.sin(x_proj), torch.cos(x_proj)], dim=-1) # [..., D, 2*num_bands]
    return x_proj.reshape(x.shape[:-1] + (-1,))

class NeRFMLP(nn.Module):
    def __init__(self, input_ch=3, input_ch_views=3, num_layers=8, hidden_dim=256, skip_indices=[4]):
        super().__init__()
        self.num_layers = num_layers
        self.skip_indices = skip_indices
        # Simplified MLP structure for clarity; original NeRF has skip connections
        self.layers = nn.ModuleList()
        for i in range(num_layers):
            in_dim = input_ch + (input_ch if i in skip_indices else 0)
            out_dim = hidden_dim
            self.layers.append(nn.Linear(in_dim, out_dim))
            if i < num_layers - 1:
                self.layers.append(nn.ReLU())
        # View direction processing
        self.view_layers = nn.ModuleList()
        self.view_layers.append(nn.Linear(input_ch_views, hidden_dim // 4))
        self.view_layers.append(nn.ReLU())
        self.view_layers.append(nn.Linear(hidden_dim // 4, hidden_dim))
        self.view_layers.append(nn.ReLU())
        self.color_layer = nn.Linear(hidden_dim, 3)
        self.sigma_layer = nn.Linear(hidden_dim, 1)

    def forward(self, x, d):
        # x: [R, N, 3], d: [R, N, 3]
        x_enc = positional_encoding(x)
        d_enc = positional_encoding(d)
        
        h = x_enc
        for i, layer in enumerate(self.layers[:-1]):
            if i in self.skip_indices:
                h = torch.cat([h, x_enc], dim=-1)
            h = layer(h)
        
        # Process view direction
        d_h = self.view_layers[0](d_enc)
        d_h = self.view_layers[1](d_h)
        d_h = self.view_layers[2](d_h)
        d_h = self.view_layers[3](d_h)
        
        h = h + d_h
        sigma = self.sigma_layer(h)
        color = self.color_layer(h)
        return sigma, color

def render_coarse_fine(rays_o, rays_d, near, far, coarse_model, fine_model, 
                       n_samples_coarse=64, n_samples_fine=128, 
                       perturb=True, lindisp=False, raw_noise_std=1.0):
    """
    Renders a batch of rays using coarse and fine sampling.
    rays_o: [R, 3]
    rays_d: [R, 3]
    near: float
    far: float
    """
    R = rays_o.shape[0]
    
    # 1. Coarse Sampling
    t_vals = torch.linspace(0., 1., n_samples_coarse).to(rays_o.device)
    if not lindisp:
        z_vals = near * (1. - t_vals) + far * t_vals
    else:
        z_vals = 1. / (1. / near * (1. - t_vals) + 1. / far * t_vals)
    
    z_vals = z_vals.expand(R, -1) # [R, N_coarse]
    
    if perturb:
        mids = 0.5 * (z_vals[:, 1:] + z_vals[:, :-1])
        upper = torch.cat([mids, z_vals[:, -1:]], -1)
        lower = torch.cat([z_vals[:, :1], mids], -1)
        t_rand = torch.rand_like(z_vals)
        z_vals = lower + (upper - lower) * t_rand
    
    # 2. Coarse Query
    pts = rays_o[:, None, :] + rays_d[:, None, :] * z_vals[..., None] # [R, N, 3]
    # Assume viewdirs are same for all samples on a ray
    viewdirs = rays_d[:, None, :].expand(-1, n_samples_coarse, -1)
    
    sigma_raw_coarse, color_raw_coarse = coarse_model(pts.reshape(-1, 3), viewdirs.reshape(-1, 3))
    sigma_raw_coarse = sigma_raw_coarse.reshape(R, n_samples_coarse, 1)
    color_raw_coarse = color_raw_coarse.reshape(R, n_samples_coarse, 3)
    
    # Add noise to density for regularization [S11, unofficial]
    sigma_raw_coarse = sigma_raw_coarse + torch.randn_like(sigma_raw_coarse) * raw_noise_std
    
    # 3. Coarse Compositing
    dists = z_vals[:, 1:] - z_vals[:, :-1] # [R, N-1]
    dists = torch.cat([dists, torch.tensor([1e10]).expand(dists.shape[0], 1).to(dists.device)], -1) # [R, N]
    dists = dists * torch.norm(rays_d, dim=-1, keepdim=True) # Scale by ray direction norm if not unit [S11, unofficial]
    
    alpha_coarse = 1. - torch.exp(-torch.relu(sigma_raw_coarse[..., 0]) * dists) # [R, N]
    transmittance_coarse = torch.cumprod(1. - alpha_coarse + 1e-10, dim=1) # [R, N]
    transmittance_coarse = torch.cat([torch.ones(R, 1).to(transmittance_coarse.device), transmittance_coarse[:, :-1]], dim=1)
    
    weights_coarse = alpha_coarse * transmittance_coarse # [R, N]
    color_coarse = torch.sum(weights_coarse[..., None] * torch.sigmoid(color_raw_coarse), dim=1) # [R, 3]
    
    # 4. Fine Sampling (Importance Sampling from Coarse Weights)
    # Sample from the distribution defined by weights_coarse (excluding last sample which has infinite dist)
    # This is a simplified implementation of sampling from CDF
    weights_coarse_sum = weights_coarse[:, :-1].sum(dim=1, keepdim=True) # [R, 1]
    pdf_coarse = weights_coarse[:, :-1] / (weights_coarse_sum + 1e-10) # [R, N-1]
    
    # Generate uniform random numbers for stratified sampling
    u = torch.rand(R, n_samples_fine).to(rays_o.device)
    
    # Inverse CDF sampling (simplified for uniform bins, requires cumulative sum)
    cdf_coarse = torch.cumsum(pdf_coarse, dim=1) # [R, N-1]
    cdf_coarse = torch.cat([torch.zeros(R, 1).to(cdf_coarse.device), cdf_coarse], dim=1) # [R, N]
    
    # Find indices where u falls in the CDF
    indices = torch.searchsorted(cdf_coarse, u, right=True) # [R, N_fine]
    # Clamp indices
    indices = torch.clamp(indices, 0, n_samples_coarse - 2)
    
    # Interpolate z_vals based on indices
    # This is a placeholder for the actual interpolation logic which depends on the specific implementation
    # For simplicity, we assume we pick the coarse z_vals at the selected indices
    # A more accurate implementation would interpolate between the bin edges
    z_vals_fine = torch.gather(z_vals[:, :-1], 1, indices) # [R, N_fine]
    
    # Concatenate and sort
    z_vals_all = torch.cat([z_vals, z_vals_fine], dim=1) # [R, N_coarse + N_fine]
    z_vals_all, sort_idx = torch.sort(z_vals_all, dim=1)
    
    # 5. Fine Query (Detached from Coarse Graph)
    pts_fine = rays_o[:, None, :] + rays_d[:, None, :] * z_vals_all[..., None]
    viewdirs_fine = rays_d[:, None, :].expand(-1, z_vals_all.shape[1], -1)
    
    # Detach to prevent gradient flow to coarse model
    pts_fine_detached = pts_fine.detach()
    viewdirs_fine_detached = viewdirs_fine.detach()
    
    sigma_raw_fine, color_raw_fine = fine_model(pts_fine_detached.reshape(-1, 3), viewdirs_fine_detached.reshape(-1, 3))
    sigma_raw_fine = sigma_raw_fine.reshape(R, z_vals_all.shape[1], 1)
    color_raw_fine = color_raw_fine.reshape(R, z_vals_all.shape[1], 3)
    
    # 6. Fine Compositing
    dists_fine = z_vals_all[:, 1:] - z_vals_all[:, :-1]
    dists_fine = torch.cat([dists_fine, torch.tensor([1e10]).expand(dists_fine.shape[0], 1).to(dists_fine.device)], -1)
    dists_fine = dists_fine * torch.norm(rays_d, dim=-1, keepdim=True)
    
    alpha_fine = 1. - torch.exp(-torch.relu(sigma_raw_fine[..., 0]) * dists_fine)
    transmittance_fine = torch.cumprod(1. - alpha_fine + 1e-10, dim=1)
    transmittance_fine = torch.cat([torch.ones(R, 1).to(transmittance_fine.device), transmittance_fine[:, :-1]], dim=1)
    
    weights_fine = alpha_fine * transmittance_fine
    color_fine = torch.sum(weights_fine[..., None] * torch.sigmoid(color_raw_fine), dim=1)
    
    return color_coarse, color_fine
```

**Edge Cases and Pitfalls:**
*   **Numerical Stability:** Add a small epsilon ($\epsilon = 10^{-10}$) to the cumulative product of transmittance to prevent underflow [S14, S25, unofficial].
*   **Last Segment:** The last segment distance must be set to a large value (e.g., $10^{10}$) to ensure the ray terminates and the last sample contributes to the color if it is opaque [S11, unofficial][S13].
*   **View Direction:** If the model uses view directions, they must be passed to the network. If not, only positions are used [S1, S25, unofficial].
*   **Depth Calculation:** Depth is computed as the weighted average of sample depths. If opacity is very low, depth can be noisy [S11, unofficial].

## Subtopics

### [NeRF Importance Sampling](1565-nerf-importance-sampling.md)
Refines sampling by drawing points from a PDF derived from coarse alpha weights, solving aliasing in high-opacity regions.

### [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md)
A modular, high-performance pipeline using occupancy grids and proposal networks to accelerate rendering, building on the coarse/fine concept.

### [VF-NeRF](3138-vf-nerf.md)
Uses viewshed fields for rigid registration between NeRFs, focusing on novel view generation for alignment rather than training efficiency.

### [Voxel-based Volume Rendering](3845-voxel-based-volume-rendering.md)
Trains a voxel representation where multiple voxels contribute to a single sample, differing from the pure MLP approach.

## Operating characteristics

*   **Time Complexity:** $O(R \cdot (N_{coarse} + N_{fine}))$ per image, where $R$ is the number of rays (pixels) and $N$ is the number of samples. Training is slow due to the need to query the MLP for every sample in every iteration [S5, S9, unofficial].
*   **Memory Complexity:** $O(R \cdot N)$ for storing intermediate tensors during the forward pass. Chunking is used to manage memory [S13].
*   **Measured Performance:** Training can take 10+ hours for a single scene on a GPU [S5, unofficial]. Rendering an image takes less than a second to ~30 seconds depending on resolution and hardware [S9, unofficial].
*   **Strengths:** High-quality novel view synthesis with view-dependent effects; differentiable end-to-end training; no need for explicit geometry (meshes/voxels) [S3][S9, unofficial].
*   **Limitations:** Slow training and inference compared to grid-based methods; prone to noise and aliasing if sampling is insufficient (hence coarse/fine) [S1, S15, unofficial]; requires dense input views for good results [S5, unofficial].
*   **When to Use:** High-fidelity view synthesis from a moderate number of posed images.
*   **When Not to Use:** Real-time applications (use [In-Shader Neural Volume Ray Marching](1495-in-shader-neural-volume-ray-marching.md) or [NerfAcc](1619-nerfacc-rendering-pipeline.md)); scenes with very sparse input views (use [DS-NeRF](5) or other sparse-view variants).

## Background

NeRF (Neural Radiance Fields) was introduced by Mildenhall et al. in 2020 [S3][S9, unofficial]. It represents a scene as a continuous volumetric radiance field parameterized by an MLP. The method was published in ECCV 2020 and received an Oral Presentation and Best Paper Honorable Mention [S9, unofficial]. It builds on the concept of implicit neural representations [S7, unofficial] and differentiable volume rendering [S1, S14, unofficial]. The coarse/fine sampling strategy was introduced to address the issue of insufficient sampling in complex scenes, leading to better convergence and higher quality renders [S15, S24, unofficial].

## Sources

[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S2] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S3] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S4] LiDAR-NeRF: Novel LiDAR View Synthesis via Neural Radiance Fields, https://arxiv.org/abs/2304.10406v2, fetched 2026-10-26
[S5] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S6] GreatDrake/reparameterized-volume-sampling, https://github.com/GreatDrake/reparameterized-volume-sampling, fetched 2026-10-26 (unofficial)
[S7] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S8] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S9] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S10] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S11] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S12] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
[S13] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S14] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S15] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S16] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S17] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S18] pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis, https://doi.org/10.1109/cvpr46437.2021.00574, fetched 2026-10-26
[S19] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
[S20] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
[S21] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
[S22] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
[S23] Improving Geometric Consistency for 360-Degree Neural Radiance Fields in Indoor Scenarios, https://arxiv.org/abs/2503.13710v1, fetched 2026-10-27
[S24] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S25] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S26] YoujiaZhang/NeMF, https://github.com/YoujiaZhang/NeMF, fetched 2026-10-27 (unofficial)
[S27] DGNR: Density-Guided Neural Point Rendering of Large Driving Scenes, https://doi.org/10.1109/tase.2024.3410891, fetched 2026-10-27
[S28] 3DGS-101/textbook/05-alpha-compositing.md at main - GitHub, https://github.com/Chamud/3DGS-101/blob/main/textbook/05-alpha-compositing.md, fetched 2026-10-28 (unofficial)
[S29] VoxNeRF: Bridging Voxel Representation and Neural Radiance Fields for ..., https://arxiv.org/abs/2311.05289, fetched 2026-10-29
[S30] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S31] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S32] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
[S33] HG3-NeRF: Hierarchical Geometric, Semantic, and Photometric Guided Neural Radiance Fields for Sparse View Inputs, https://doi.org/10.48550/arXiv.2401.11711, fetched 2026-10-29 (unofficial)
[S34] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
[S35] https://github.com/Mayxmu/PCMP-NeRF, https://github.com/Mayxmu/PCMP_NeRF, fetched 2026-10-30 (unofficial)
