# NeRF Volumetric Rendering (Coarse/Fine Sampling)

Also called: Vanilla NeRF, Standard NeRF, NeRF Rendering Equation, NeRF Two-Stage Sampling, Volume Rendering (NeRF), NeRF Coarse-to-Fine Rendering, Neural Radiance Field Rendering, coarse-fine volumetric rendering, Neural Radiance Field (NeRF) Volumetric Rendering

## Overview

NeRF Volumetric Rendering (Coarse/Fine Sampling) is a differentiable rendering method that synthesizes novel views of 3D scenes by querying a neural radiance field along camera rays and integrating the outputs using classical volume rendering techniques [S3, S14]. It solves the problem of insufficient sampling in high-opacity regions, which leads to aliasing and noise, by employing a two-stage sampling strategy: a coarse network samples uniformly (or with stratified sampling) to estimate the density distribution, and a fine network samples more densely in high-density regions identified by the coarse stage to produce a refined estimate [S1, S8, S13, S22, unofficial]. It sets itself apart from its siblings, such as [Iterative Ray Marching with Neural Volume Inference](1494-iterative-ray-marching-with-neural-volume-inference.md) which uses iterative refinement, and [DVGO Volume Rendering Forward Pass](2363-dvgo-volume-rendering-forward-pass.md) which uses voxel grids, by relying on a continuous MLP representation with a specific coarse-to-fine hierarchical sampling process [S1, S8, S13, unofficial].

## How it works

**Inputs and Outputs:**
*   **Inputs:** A batch of rays, where each ray is defined by an origin $\mathbf{o}$, a direction $\mathbf{d}$, and near/far bounds $t_n, t_f$. The rays are typically derived from camera poses and pixel coordinates. Optional viewing directions $\mathbf{d}$ are included if view-dependent color is modeled.
*   **Outputs:** For each ray, the final RGB color $\hat{\mathbf{C}}$, accumulated opacity $\alpha_{acc}$, and optionally depth $\hat{D}$ and disparity.

**Data Structures:**
*   **Ray Batch:** A tensor of shape $[R, 8]$ or $[R, 11]$ (if viewdirs are included), where $R$ is the number of rays. Columns typically contain: `rays_o` (3), `rays_d` (3), `viewdirs` (3, optional), `near` (1), `far` (1).
*   **Sample Positions:** A tensor `z_vals` of shape $[R, N]$ containing the sampled distances $t$ along the ray, where $N$ is the number of samples.
*   **3D Points:** A tensor `pts` of shape $[R, N, 3]$ containing the world coordinates $\mathbf{p}_i = \mathbf{o} + t_i \mathbf{d}$.
*   **Network Output:** A tensor of shape $[R, N, 4]$ containing raw density $\sigma$ (1 channel) and raw color $\mathbf{c}$ (3 channels) for each sample.

**Parameters:**
*   `N_samples` (or `n_samples_coarse`): Number of coarse samples per ray. Default: 64 [S6, S8, S9, S23, unofficial].
*   `N_importance` (or `n_samples_fine`): Number of additional fine samples per ray. Default: 0 (disabled) [S8, S9, S23, unofficial].
*   `perturb`: Float controlling jitter. 0.0 for no jitter, 1.0 for full jitter. Default: 1.0 [S8, S9, S23, unofficial].
*   `lindisp`: Boolean. If True, sample linearly in inverse depth (disparity) rather than depth. Default: False [S8, S7, S23, unofficial].
*   `raw_noise_std`: Standard deviation of Gaussian noise added to raw density predictions for regularization. Default: 0.0, Recommended: 1.0 [S9, S8, S23, unofficial].
*   `white_bkgd`: Boolean. If True, composites result onto a white background. Default: False [S9, S7, S23, unofficial].
*   `chunk`: Number of rays processed in parallel to manage memory. Default: 32768 [S23, unofficial].

**Steps:**

1.  **Ray Generation and Discretization (Coarse):**
    For each ray, generate $N_{coarse}$ sample distances $t_{val}$ linearly spaced between 0 and 1. Map these to the near/far bounds $[t_n, t_f]$.
    If `lindisp` is False:
    $$ t_i = t_n (1 - t_{val}) + t_f t_{val} $$
    If `lindisp` is True:
    $$ t_i = \frac{1}{\frac{1}{t_n}(1 - t_{val}) + \frac{1}{t_f}t_{val}} $$
    If `perturb` > 0, apply stratified sampling: compute midpoints $m_i = 0.5(t_{i+1} + t_i)$, define intervals $[t_i, m_i]$ and $[m_i, t_{i+1}]$, and jitter $t_i$ uniformly within these intervals [S6, S8, S23, unofficial].

2.  **Neural Query (Coarse):**
    Compute 3D points $\mathbf{p}_i = \mathbf{o} + t_i \mathbf{d}$. Query the coarse MLP $F_{\theta_{coarse}}$ with $\mathbf{p}_i$ (and $\mathbf{d}$ if `use_viewdirs` is True) to get raw density $\sigma_i$ and raw color $\mathbf{c}_i$.
    Apply noise to density: $\sigma_i' = \sigma_i + \mathcal{N}(0, \text{raw\_noise\_std})$ [S9, S23, unofficial].
    Apply activation functions:
    $$ \alpha_i = 1 - \exp(-\text{ReLU}(\sigma_i') \delta_i) $$
    $$ \mathbf{c}_i = \text{sigmoid}(\mathbf{c}_i) $$
    where $\delta_i = t_{i+1} - t_i$ is the distance between samples. The last segment $\delta_N$ is set to a large value (e.g., $10^{10}$) to ensure the ray terminates [S9, unofficial][S11][S24, unofficial].

3.  **Compositing (Coarse):**
    Compute transmittance $T_i$ and weights $w_i$:
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j' \delta_j\right), \quad T_1 = 1 $$
    $$ w_i = T_i (1 - \exp(-\sigma_i' \delta_i)) $$
    Compute coarse color, depth, and opacity:
    $$ \hat{\mathbf{C}}_{coarse} = \sum_{i=1}^{N_{coarse}} w_i \mathbf{c}_i $$
    $$ \hat{D}_{coarse} = \sum_{i=1}^{N_{coarse}} w_i t_i $$
    $$ \alpha_{acc} = \sum_{i=1}^{N_{coarse}} w_i $$
    If `white_bkgd` is True, add $(1 - \alpha_{acc})$ to the RGB color [S1, S9, S23, unofficial].

4.  **Importance Sampling (Fine):**
    If `N_importance` > 0, use the coarse weights $w_i$ (excluding the first sample to avoid bias) as a probability density function (PDF) to draw $N_{fine}$ additional samples [S1, unofficial][S2][S22, unofficial].
    *   Normalize weights: $p_i = w_i / \sum w_i$.
    *   Compute cumulative distribution function (CDF).
    *   Sample $N_{fine}$ indices from this CDF.
    *   Interpolate $t$ values between the coarse sample positions corresponding to the selected indices [S22, S23, unofficial].
    *   Concatenate the coarse $t$ values and the new fine $t$ values, then sort them to interleave fine samples among coarse ones [S22, unofficial].

5.  **Neural Query (Fine):**
    Query the fine MLP $F_{\theta_{fine}}$ with the new set of 3D points (and viewing directions). The fine network is independent of the coarse network [S22, S23, unofficial].
    Apply the same activation functions as in Step 2 to get $\alpha_i$ and $\mathbf{c}_i$ for all samples (coarse + fine) [S22, unofficial].

6.  **Compositing (Fine):**
    Recompute transmittance $T_i$ and weights $w_i$ for the combined set of samples using the fine network's density predictions [S22, unofficial].
    Compute fine color, depth, and opacity:
    $$ \hat{\mathbf{C}}_{fine} = \sum_{i=1}^{N_{coarse} + N_{fine}} w_i \mathbf{c}_i $$
    $$ \hat{D}_{fine} = \sum_{i=1}^{N_{coarse} + N_{fine}} w_i t_i $$
    $$ \alpha_{acc} = \sum_{i=1}^{N_{coarse} + N_{fine}} w_i $$
    Apply background compositing if `white_bkgd` is True [S22, S23, unofficial].

7.  **Loss Computation:**
    Compute the photometric loss (MSE) between the rendered color ($\hat{\mathbf{C}}_{coarse}$ or $\hat{\mathbf{C}}_{fine}$) and the ground truth RGB color $\mathbf{C}_{gt}$ [S13, unofficial][S18].
    $$ \mathcal{L}_{RGB} = \|\hat{\mathbf{C}} - \mathbf{C}_{gt}\|^2 $$
    Both coarse and fine losses can be summed for optimization [S13, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def raw2alpha(raw, dists, act_fn=F.relu):
    """Convert raw density to alpha."""
    # raw: [R, N], dists: [R, N]
    # Add epsilon for numerical stability
    alpha = 1. - torch.exp(-act_fn(raw) * dists)
    return alpha

def render_rays(rays_o, rays_d, near, far, n_samples_coarse, n_samples_fine, 
                perturb, lindisp, raw_noise_std, white_bkgd, 
                coarse_model, fine_model, use_viewdirs):
    """
    Render a batch of rays using coarse/fine sampling.
    rays_o: [R, 3]
    rays_d: [R, 3]
    near: [R]
    far: [R]
    """
    R = rays_o.shape[0]
    
    # 1. Coarse Sampling
    t_vals = torch.linspace(0., 1., n_samples_coarse, device=rays_o.device)
    if not lindisp:
        z_vals = near.unsqueeze(-1) * (1. - t_vals) + far.unsqueeze(-1) * t_vals
    else:
        z_vals = 1. / (1. / near.unsqueeze(-1) * (1. - t_vals) + 1. / far.unsqueeze(-1) * t_vals)
        
    if perturb > 0.:
        mids = .5 * (z_vals[..., 1:] + z_vals[..., :-1])
        upper = torch.cat([mids, z_vals[..., -1:]], -1)
        lower = torch.cat([z_vals[..., :1], mids], -1)
        t_rand = torch.rand_like(z_vals)
        z_vals = lower + (upper - lower) * t_rand
        
    # Expand for broadcasting
    z_vals = z_vals.expand(R, n_samples_coarse)
    
    # 2. Coarse Query
    pts = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals.unsqueeze(-1) # [R, N, 3]
    
    # Flatten for network query
    pts_flat = pts.reshape(-1, 3)
    viewdirs_flat = rays_d.unsqueeze(1).expand(-1, n_samples_coarse, -1).reshape(-1, 3) if use_viewdirs else None
    
    # Query coarse model
    raw_coarse = coarse_model(pts_flat, viewdirs_flat) # [R*N, 4]
    raw_coarse = raw_coarse.reshape(R, n_samples_coarse, 4)
    
    sigma_coarse = raw_coarse[..., 0]
    rgb_coarse = raw_coarse[..., 1:]
    
    # Add noise to sigma
    if raw_noise_std > 0.:
        sigma_coarse = sigma_coarse + torch.randn_like(sigma_coarse) * raw_noise_std
        
    # Compute dists
    dists = z_vals[..., 1:] - z_vals[..., :-1]
    dists = torch.cat([dists, torch.tensor([1e10], device=rays_o.device).expand(R, 1)], -1)
    # Scale by ray direction norm if necessary (assuming unit vectors in many implementations, but safe to scale)
    dists = dists * torch.norm(rays_d.unsqueeze(-1), dim=-1)
    
    # Compute alpha and weights
    alpha_coarse = raw2alpha(sigma_coarse, dists)
    T_coarse = torch.cumprod(1. - alpha_coarse + 1e-10, dim=-1) # [R, N]
    T_coarse = torch.cat([torch.ones_like(T_coarse[..., :1]), T_coarse[..., :-1]], -1) # Shift right, T_1 = 1
    weights_coarse = T_coarse * alpha_coarse
    
    # Coarse Color
    rgb_coarse = torch.sigmoid(rgb_coarse)
    rgb_coarse = (weights_coarse.unsqueeze(-1) * rgb_coarse).sum(dim=1)
    
    # Coarse Depth
    depth_coarse = (weights_coarse * z_vals).sum(dim=1)
    
    # Accumulated Opacity
    acc_coarse = weights_coarse.sum(dim=1)
    
    # 3. Fine Sampling (if enabled)
    if n_samples_fine > 0:
        # Sample from coarse weights (excluding first sample)
        weights_coarse_no_first = weights_coarse[..., 1:]
        pdf_coarse = weights_coarse_no_first / (weights_coarse_no_first.sum(dim=-1, keepdim=True) + 1e-10)
        
        # Sample indices
        # Note: In practice, this involves inverse CDF sampling. 
        # Simplified representation: draw N_fine samples from the distribution defined by pdf_coarse
        # and interpolate z_vals.
        # For implementation completeness, we assume a helper `sample_pdf` exists or implement it:
        
        bins = z_vals # [R, N_coarse]
        # Compute CDF
        cdf = torch.cumsum(weights_coarse_no_first, dim=-1)
        cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], dim=-1)
        
        # Uniform samples for stratified sampling within bins
        u = torch.rand(R, n_samples_fine, device=rays_o.device)
        # Inverse CDF lookup (simplified, assuming sorted bins)
        # In real code, torch.searchsorted is used.
        # Here we assume a function `sample_pdf` that returns z_vals_fine
        z_vals_fine = sample_pdf(z_vals[..., 1:], weights_coarse_no_first, n_samples_fine, detach=True)
        
        # Concatenate and sort
        z_vals_all = torch.cat([z_vals, z_vals_fine], dim=-1)
        z_vals_all, _ = torch.sort(z_vals_all, dim=-1)
        
        # Re-query fine model
        pts_all = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * z_vals_all.unsqueeze(-1)
        pts_all_flat = pts_all.reshape(-1, 3)
        viewdirs_all_flat = rays_d.unsqueeze(1).expand(-1, z_vals_all.shape[1], -1).reshape(-1, 3) if use_viewdirs else None
        
        raw_fine = fine_model(pts_all_flat, viewdirs_all_flat)
        raw_fine = raw_fine.reshape(R, z_vals_all.shape[1], 4)
        
        sigma_fine = raw_fine[..., 0]
        rgb_fine = raw_fine[..., 1:]
        
        # Re-compute dists for all samples
        dists_all = z_vals_all[..., 1:] - z_vals_all[..., :-1]
        dists_all = torch.cat([dists_all, torch.tensor([1e10], device=rays_o.device).expand(R, 1)], -1)
        dists_all = dists_all * torch.norm(rays_d.unsqueeze(-1), dim=-1)
        
        alpha_fine = raw2alpha(sigma_fine, dists_all)
        T_fine = torch.cumprod(1. - alpha_fine + 1e-10, dim=-1)
        T_fine = torch.cat([torch.ones_like(T_fine[..., :1]), T_fine[..., :-1]], -1)
        weights_fine = T_fine * alpha_fine
        
        # Fine Color
        rgb_fine = torch.sigmoid(rgb_fine)
        rgb_fine = (weights_fine.unsqueeze(-1) * rgb_fine).sum(dim=1)
        
        # Fine Depth
        depth_fine = (weights_fine * z_vals_all).sum(dim=1)
        
        acc_fine = weights_fine.sum(dim=1)
        
        return {
            'rgb_coarse': rgb_coarse,
            'depth_coarse': depth_coarse,
            'acc_coarse': acc_coarse,
            'rgb_fine': rgb_fine,
            'depth_fine': depth_fine,
            'acc_fine': acc_fine
        }
    else:
        return {
            'rgb_coarse': rgb_coarse,
            'depth_coarse': depth_coarse,
            'acc_coarse': acc_coarse
        }

def sample_pdf(z_vals, weights, N_samples, detach=False):
    """
    Stratified sampling from the distribution defined by weights.
    z_vals: [R, N_coarse]
    weights: [R, N_coarse-1]
    Returns: [R, N_samples]
    """
    # Get weights for bins (N_coarse-1 bins for N_coarse points)
    # weights are already [R, N_coarse-1]
    
    # Compute CDF
    cdf = torch.cumsum(weights, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], dim=-1)
    
    # Normalize CDF to [0, 1]
    cdf = cdf / cdf[..., -1:].expand_as(cdf)
    
    # Generate uniform samples
    u = torch.rand(z_vals.shape[0], N_samples, device=z_vals.device)
    
    # Inverse CDF
    # Find indices where cdf < u
    # torch.searchsorted is efficient
    indices = torch.searchsorted(cdf, u, side='left')
    
    # Clip indices to valid range
    indices = torch.clamp(indices, 0, z_vals.shape[1] - 2)
    
    # Interpolate
    z0 = z_vals[torch.arange(z_vals.shape[0]), indices]
    z1 = z_vals[torch.arange(z_vals.shape[0]), indices + 1]
    
    # Get CDF values at indices
    cdf0 = cdf[torch.arange(cdf.shape[0]), indices]
    cdf1 = cdf[torch.arange(cdf.shape[0]), indices + 1]
    
    # Linear interpolation
    t = (u - cdf0) / (cdf1 - cdf0 + 1e-10)
    z_samples = z0 + t * (z1 - z0)
    
    if detach:
        z_samples = z_samples.detach()
        
    return z_samples
```

**Edge Cases and Pitfalls:**
*   **Numerical Stability:** Small epsilon ($10^{-10}$) is added to transmittance calculations to prevent `NaN` gradients when alpha is 1 or 0 [S12, S23, unofficial].
*   **Last Segment:** The last segment distance must be large (e.g., $10^{10}$) to ensure the ray terminates and the accumulated opacity can reach 1.0 [S9, unofficial][S11].
*   **View Direction:** If `use_viewdirs` is False, the network input is only 3D position, resulting in view-independent color [S23, unofficial].
*   **Memory:** Processing all rays at once can exceed GPU memory. Rays should be processed in chunks (`chunk` parameter) [S11][S12, S23, unofficial].

## Subtopics

### [NeRF Importance Sampling](1565-nerf-importance-sampling.md)
NeRF Importance Sampling is a two-stage volume rendering strategy that refines the discrete sampling distribution along a ray by drawing additional points from a probability density function (PDF) derived from the coarse stage's opacity weights. It solves the problem of insufficient sampling in high-opacity regions, which leads to aliasing and noise, by concentrating fine samples where the volume contributes most to the final color. It sets itself apart from its siblings, such as [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md) which uses learned acceleration structures, and [VF-NeRF](3138-vf-nerf.md) which uses view-dependent feature grids, by relying on a simple, differentiable stratified resampling of the coarse alpha weights without auxiliary networks or complex data structures.

### [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md)
The NerfAcc Rendering Pipeline is a modular, high-performance volumetric rendering method implemented as a PyTorch library. It decouples the sampling strategy from the radiance field representation, supporting both Occupancy Grids and Proposal Networks to accelerate training and inference. Unlike standard uniform sampling, it uses interval-based ray representation to support conical frustums and integrates coarse/fine sampling with importance sampling to concentrate samples on surfaces while skipping empty space. It builds on [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md) by providing a unified API for efficient transmittance estimation and differentiable rendering.

### [VF-NeRF](3138-vf-nerf.md)
VF-NeRF is a method for rigid registration between two Neural Radiance Fields (NeRFs) when the original camera positions are unknown. It introduces **Viewshed Fields (VF)**, an implicit function that maps an oriented point (3D location and viewing direction) to a scalar indicating how well that point was covered by the original camera set. By using Normalizing Flows to generate high-scoring oriented points, VF-NeRF creates informative virtual camera viewpoints for photometric alignment, avoiding the noise and poor coverage associated with random sampling. It differs from [NeRF Importance Sampling](1565-nerf-importance-sampling.md) by focusing on registration via novel view generation rather than training efficiency, and from [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md) by its specific application to inter-NeRF alignment.

### Voxel-based Volume Rendering
A volume rendering technique that trains a voxel representation of a radiance field, where multiple voxels contribute to a single sample during rendering [S1, unofficial]. It differs from voxelization (sparse octree) in that voxel-based rendering uses dense or semi-dense voxels where multiple voxels contribute to a single sample, whereas voxelization creates a sparse octree containing voxels only where the model determines the space is solid [S1, unofficial]. It is used as a coarse model for two-tier sampling in NeRF-like architectures to determine opaque regions in space [S1, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(R \cdot (N_{coarse} + N_{fine}))$ neural network queries per batch, where $R$ is the number of rays and $N$ is the number of samples. The fine stage doubles the query cost if $N_{fine} \approx N_{coarse}$.
*   **Memory Complexity:** $O(R \cdot N)$ for storing sample positions, densities, and colors. Chunking reduces peak memory usage at the cost of increased kernel launch overhead.
*   **Measured Performance:** Training typically takes 10-20 hours on a single GPU for high-resolution scenes (e.g., 512x512) [S7, unofficial]. Rendering a single image can take seconds to minutes depending on resolution and $N$ [S7, unofficial].
*   **Scaling:** Performance scales linearly with the number of samples and rays. Increasing $N_{fine}$ improves quality but increases compute.
*   **Strengths:** High-quality novel view synthesis; differentiable end-to-end training; handles complex geometry and appearance (view-dependent effects).
*   **Limitations:** Slow training and rendering compared to grid-based methods; requires many input views for good reconstruction; prone to noise in sparse regions without importance sampling.
*   **When to Use:** When high-fidelity novel views are required and computational resources allow for long training times.
*   **When Not to Use:** For real-time applications or scenes with very sparse input views where grid-based methods (like [DVGO](2363-dvgo-volume-rendering-forward-pass.md)) may be more efficient.

## Background

NeRF (Neural Radiance Fields) was introduced by Mildenhall et al. in 2020 [S3]. It represents a 3D scene as a continuous volumetric function modeled by a multi-layer perceptron (MLP) that maps 3D coordinates and viewing directions to volume density and RGB color. The method uses classical volume rendering equations to integrate these predictions along camera rays, making the entire pipeline differentiable. This allows the network to be optimized using gradient descent on a set of input images with known camera poses [S3][S13, unofficial]. The coarse/fine sampling strategy was introduced to address the issue of uniform sampling being inefficient, leading to noisy renders in complex scenes [S1, S8, unofficial].

## Sources

[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S2] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S3] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S4] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S5] GreatDrake/reparameterized-volume-sampling, https://github.com/GreatDrake/reparameterized-volume-sampling, fetched 2026-10-26 (unofficial)
[S6] Yonghao-Lee/differentiable-rendering, https://github.com/Yonghao-Lee/differentiable-rendering, fetched 2026-10-26 (unofficial)
[S7] bmild/nerf, https://github.com/bmild/nerf, fetched 2026-10-26 (unofficial)
[S8] yenchenlin/nerf-pytorch, https://github.com/yenchenlin/nerf-pytorch, fetched 2026-10-26 (unofficial)
[S9] yashbhalgat/HashNeRF-pytorch, https://github.com/yashbhalgat/HashNeRF-pytorch, fetched 2026-10-26 (unofficial)
[S10] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
[S11] learning3d/assignment3, https://github.com/learning3d/assignment3, fetched 2026-10-26
[S12] samirangode/Volume-Rendering-and-Neural-Radiance-Fields, https://github.com/samirangode/Volume-Rendering-and-Neural-Radiance-Fields, fetched 2026-10-26 (unofficial)
[S13] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S14] 3D volumetric rendering with NeRF - Keras, https://keras.io/examples/vision/nerf/, fetched 2026-10-26
[S15] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
[S16] pi-GAN: Periodic Implicit Generative Adversarial Networks for 3D-Aware Image Synthesis, https://doi.org/10.1109/cvpr46437.2021.00574, fetched 2026-10-26
[S17] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
[S18] SinNeRF: Training Neural Radiance Fields on Complex Scenes from a Single Image, https://doi.org/10.1007/978-3-031-20047-2_42, fetched 2026-10-26
[S19] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
[S20] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
[S21] Improving Geometric Consistency for 360-Degree Neural Radiance Fields in Indoor Scenarios, https://arxiv.org/abs/2503.13710v1, fetched 2026-10-27
[S22] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S23] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S24] 3DGS-101/textbook/05-alpha-compositing.md at main - GitHub, https://github.com/Chamud/3DGS-101/blob/main/textbook/05-alpha-compositing.md, fetched 2026-10-28 (unofficial)
[S25] VoxNeRF: Bridging Voxel Representation and Neural Radiance Fields for ..., https://arxiv.org/abs/2311.05289, fetched 2026-10-29
[S26] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S27] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S28] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
[S29] TöRF: Time-of-Flight Radiance Fields for Dynamic Scene View Synthesis, https://doi.org/10.48550/arxiv.2109.15271, fetched 2026-10-29 (unofficial)
