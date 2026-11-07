# NeRF Volumetric Rendering (Coarse/Fine Sampling)

Also called: Vanilla NeRF, Standard NeRF, NeRF Volume Rendering, NeRF Two-Stage Sampling, Volume Rendering (NeRF), NeRF Coarse-to-Fine Rendering, Neural Radiance Fields rendering, coarse-fine volumetric rendering

## Overview

NeRF Volumetric Rendering with Coarse/Fine Sampling is a differentiable rendering method that synthesizes novel views of 3D scenes by querying a neural radiance field (MLP) along camera rays and integrating the outputs using classical volume rendering techniques [S3, S14]. It solves the problem of low-resolution convergence and inefficient sampling in single-stage neural volume rendering by employing a two-stage sampling strategy: a coarse network samples uniformly to identify opaque regions, and a fine network samples densely in those regions to refine the image [S8, S13, unofficial]. What sets it apart from its siblings, such as [Iterative Ray Marching with Neural Volume Inference](1494-iterative-ray-marching-with-neural-volume-inference.md) or [DVGO Volume Rendering Forward Pass](2363-dvgo-volume-rendering-forward-pass.md), is the explicit use of two distinct MLPs (coarse and fine) with independent weights, where the coarse stage provides a probability distribution for resampling the fine stage, rather than using a single network with hierarchical data structures or iterative refinement loops [S13, S18, unofficial].

## How it works

The method builds on [Direct volume ray casting](19-direct-volume-ray-casting.md) but replaces the discrete volume texture lookup with a neural network query. The core mechanism follows the differentiable volume rendering integral [S1, unofficial][S14].

### Inputs and Outputs

**Inputs:**
*   `rays_o`: Tensor of shape `[R, 3]`, ray origins in world space [S6, S8, unofficial].
*   `rays_d`: Tensor of shape `[R, 3]`, unnormalized ray directions (Blender convention: camera looks down -z) [S6, S8, unofficial].
*   `near`, `far`: Scalars defining the near and far clipping planes along the ray parameter $t$ [S5, S8, unofficial].
*   `coarse_model`: An MLP $F_{\theta_c}$ mapping 5D coordinates to density and color [S13, S18, unofficial].
*   `fine_model`: An MLP $F_{\theta_f}$ mapping 5D coordinates to density and color [S13, S18, unofficial].
*   `N_samples_coarse`: Integer, number of samples per ray for the coarse stage (default 64) [S8, S19, unofficial].
*   `N_importance`: Integer, number of additional samples per ray for the fine stage (default 128) [S8, S19, unofficial].
*   `perturb`: Float, amount of stratified noise added to sampling (default 1.0) [S7, S19, unofficial].
*   `lindisp`: Boolean, if True, sample linearly in inverse depth (disparity) rather than depth (default False) [S8, S19, unofficial].
*   `use_viewdirs`: Boolean, if True, include viewing direction in the MLP input (default False) [S19, unofficial].

**Outputs:**
*   `rgb_coarse`: Tensor of shape `[R, 3]`, rendered RGB color from coarse stage [S18, unofficial].
*   `rgb_fine`: Tensor of shape `[R, 3]`, rendered RGB color from fine stage [S18, unofficial].
*   `acc_coarse`, `acc_fine`: Tensors of shape `[R]`, accumulated opacity [S18, unofficial].
*   `depth_coarse`, `depth_fine`: Tensors of shape `[R]`, rendered depth [S18, unofficial].

### Data Structures

**Neural Network Input/Output:**
*   The MLP input is a concatenated vector of positional encoding for position $\gamma(\mathbf{x})$ and optionally viewing direction $\gamma(\mathbf{d})$.
*   Positional encoding $\gamma(\mathbf{x})$ maps $\mathbf{x} \in \mathbb{R}^3$ to a higher dimensional space using sine and cosine functions up to frequency $2^{L-1}$:
    $$ \gamma(\mathbf{x}) = [\sin(2^0 \pi \mathbf{x}), \cos(2^0 \pi \mathbf{x}), \dots, \sin(2^{L-1} \pi \mathbf{x}), \cos(2^{L-1} \pi \mathbf{x})]^T $$
    where $L$ is the number of frequency bands (default 10 for position, 4 for direction) [S2, S15][S19, unofficial].
*   The MLP output `raw` is a tensor of shape `[num_rays, num_samples, 4]`. The last dimension contains `[RGB, density]` [S1, S7, S9, unofficial].

**Ray Sampling Arrays:**
*   `z_vals`: Tensor of shape `[R, N_samples]`, representing the depth positions $t$ along the ray where samples are taken [S9, unofficial].

### Numbered Steps

1.  **Ray Preparation:** For each ray $r(t) = \mathbf{o} + t\mathbf{d}$, generate $N_{samples\_coarse}$ sample positions $t$ linearly spaced between `near` and `far`. If `lindisp` is True, interpolate in inverse depth:
    $$ t_{vals} = \text{linspace}(0, 1, N_{samples\_coarse}) $$
    $$ z_{vals} = \frac{1}{\frac{1}{\text{near}}(1 - t_{vals}) + \frac{1}{\text{far}}t_{vals}} $$
    Otherwise, interpolate linearly in depth:
    $$ z_{vals} = \text{near}(1 - t_{vals}) + \text{far} \cdot t_{vals} $$
    If `perturb` > 0, apply stratified sampling: compute midpoints $mids = 0.5(z_{vals}[:, 1:] + z_{vals}[:, :-1])$, define lower bounds $[z_{vals}[:, :1], mids]$ and upper bounds $[mids, z_{vals}[:, -1:]]$, and jitter: $z_{vals} = lower + (upper - lower) \cdot \text{rand}()$ [S5, S6, S8, unofficial].

2.  **Coarse Network Query:**
    *   Compute 3D points: $\mathbf{p}_i = \mathbf{o} + z_{vals} \cdot \mathbf{d}$ [S5, S8, unofficial].
    *   If `use_viewdirs` is True, compute viewing directions $\mathbf{d}_i = \mathbf{d}$ for each point.
    *   Apply positional encoding to $\mathbf{p}_i$ and $\mathbf{d}_i$ (if applicable).
    *   Pass encoded inputs to `coarse_model` to get `raw_coarse` of shape `[R, N_samples_coarse, 4]` [S1, S7, S9, unofficial].
    *   Split `raw_coarse` into `rgb_coarse_raw` (first 3 channels) and `sigma_coarse_raw` (4th channel) [S1, S9, unofficial].
    *   Apply activation functions: `rgb_coarse = sigmoid(rgb_coarse_raw)`, `sigma_coarse = softplus(sigma_coarse_raw)` [S1, S9, unofficial]. Note: Some implementations use ReLU for sigma and add noise with standard deviation `raw_noise_std` (default 1.0) to `sigma_coarse` before alpha conversion for regularization [S7, S9, S13, unofficial].

3.  **Coarse Volume Rendering:**
    *   Compute segment distances: $\delta_i = z_{vals}[:, i+1] - z_{vals}[:, i]$ for $i=0 \dots N-2$. Append a large value (e.g., $10^{10}$) for the last segment to ensure the ray terminates [S9, unofficial][S11].
    *   Compute alpha (opacity) for each sample:
        $$ \alpha_i = 1 - \exp(-\sigma_i \cdot \delta_i) $$
        [S6, S18, S20, unofficial].
    *   Compute transmittance $T_i$ (probability of ray passing through previous samples):
        $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
        [S12, unofficial][S24]. Initialize $T_0 = 1$.
    *   Compute weights $w_i = T_i \alpha_i$ [S6, S18, S20, unofficial].
    *   Render coarse color:
        $$ C_{coarse} = \sum_{i=1}^{N} w_i \mathbf{c}_i $$
        [S1, S12, S18, unofficial].
    *   Render coarse depth:
        $$ D_{coarse} = \sum_{i=1}^{N} w_i z_{vals, i} $$
        [S12, S18, unofficial].
    *   Render coarse accumulation (opacity):
        $$ A_{coarse} = \sum_{i=1}^{N} w_i $$
        [S18, unofficial].

4.  **Fine Sampling (Resampling):**
    *   Use the coarse weights $w_i$ (excluding the first sample if necessary, or all) as a probability distribution to sample $N_{importance}$ additional points [S1, S18, unofficial].
    *   Sample $z_{fine}$ from the discrete distribution defined by $w_i$. This concentrates samples in regions of high opacity [S1, S18, unofficial].
    *   Concatenate coarse $z_{vals}$ and fine $z_{fine}$ along the last dimension and sort them to interleave fine samples among coarse ones [S18, unofficial].
    *   Detach $z_{fine}$ from the computation graph to prevent gradients from flowing back to the coarse network during fine sampling [S18, unofficial].

5.  **Fine Network Query:**
    *   Compute 3D points for the combined sorted $z_{vals}$: $\mathbf{p}_i = \mathbf{o} + z_{vals} \cdot \mathbf{d}$ [S5, S8, unofficial].
    *   Apply positional encoding.
    *   Pass encoded inputs to `fine_model` to get `raw_fine` of shape `[R, N_samples_coarse + N_importance, 4]` [S1, S7, S9, unofficial].
    *   Split and activate `raw_fine` to get `rgb_fine` and `sigma_fine` [S1, S9, unofficial].

6.  **Fine Volume Rendering:**
    *   Compute segment distances $\delta_i$ for the combined sorted $z_{vals}$ [S9, unofficial][S11].
    *   Compute alpha $\alpha_i = 1 - \exp(-\sigma_i \cdot \delta_i)$ [S6, S18, S20, unofficial].
    *   Compute transmittance $T_i$ and weights $w_i = T_i \alpha_i$ [S6, S18, S20, unofficial].
    *   Render fine color:
        $$ C_{fine} = \sum_{i=1}^{N} w_i \mathbf{c}_i $$
        [S1, S12, S18, unofficial].
    *   Render fine depth:
        $$ D_{fine} = \sum_{i=1}^{N} w_i z_{vals, i} $$
        [S12, S18, unofficial].
    *   Render fine accumulation:
        $$ A_{fine} = \sum_{i=1}^{N} w_i $$
        [S18, unofficial].

7.  **Background Compositing (Optional):**
    *   If `white_bkgd` is True, composite the result onto a white background:
        $$ C_{final} = C_{fine} + (1 - A_{fine}) \cdot 1.0 $$
        [S6, S9, S13, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def positional_encoding(x, num_bands):
    """Apply positional encoding to input x."""
    freq_bands = 2.0 ** torch.linspace(0, num_bands - 1, num_bands).to(x.device)
    encoded = torch.cat([torch.sin(x * freq_bands), torch.cos(x * freq_bands)], dim=-1)
    return encoded

def raw2alpha(raw, dists, act_fn=F.relu):
    """Convert raw density to alpha."""
    sigma = act_fn(raw)
    alpha = 1.0 - torch.exp(-sigma * dists)
    return alpha

def render_coarse_fine(rays_o, rays_d, near, far, coarse_model, fine_model,
                       N_samples_coarse, N_importance, perturb=1.0, lindisp=False,
                       use_viewdirs=False, raw_noise_std=1.0, white_bkgd=False):
    """
    Render a batch of rays using coarse and fine sampling.
    rays_o: [R, 3]
    rays_d: [R, 3]
    """
    R = rays_o.shape[0]
    
    # 1. Coarse Sampling
    t_vals = torch.linspace(0., 1., N_samples_coarse).to(rays_o.device)
    if lindisp:
        z_vals = 1. / (1. / near * (1. - t_vals) + 1. / far * t_vals)
    else:
        z_vals = near * (1. - t_vals) + far * t_vals
    
    # Stratified sampling
    if perturb > 0.:
        mids = .5 * (z_vals[..., 1:] + z_vals[..., :-1])
        upper = torch.cat([mids, z_vals[..., -1:]], -1)
        lower = torch.cat([z_vals[..., :1], mids], -1)
        t_rand = torch.rand(z_vals.shape).to(rays_o.device)
        z_vals = lower + (upper - lower) * t_rand
    
    # Expand to [R, N]
    z_vals = z_vals.expand([R, N_samples_coarse])
    
    # Compute 3D points
    pts = rays_o[..., None, :] + rays_d[..., None, :] * z_vals[..., :, None] # [R, N, 3]
    
    # Query Coarse Model
    if use_viewdirs:
        viewdirs = rays_d[..., None, :].expand(pts.shape) # [R, N, 3]
        pts_flat = pts.reshape(-1, 3)
        viewdirs_flat = viewdirs.reshape(-1, 3)
        pts_enc = positional_encoding(pts_flat, 10)
        viewdirs_enc = positional_encoding(viewdirs_flat, 4)
        raw_coarse = coarse_model(pts_enc, viewdirs_enc) # [R*N, 4]
        raw_coarse = raw_coarse.reshape(R, N_samples_coarse, 4)
    else:
        pts_flat = pts.reshape(-1, 3)
        pts_enc = positional_encoding(pts_flat, 10)
        raw_coarse = coarse_model(pts_enc) # [R*N, 4]
        raw_coarse = raw_coarse.reshape(R, N_samples_coarse, 4)
        
    rgb_coarse_raw, sigma_coarse_raw = raw_coarse[..., :3], raw_coarse[..., 3:]
    
    # Add noise to sigma for regularization
    sigma_coarse_raw = sigma_coarse_raw + torch.randn_like(sigma_coarse_raw) * raw_noise_std
    
    # 2. Coarse Rendering
    dists = z_vals[..., 1:] - z_vals[..., :-1]
    dists = dists * torch.norm(rays_d[..., None, :], dim=-1) # Scale by ray direction norm
    dists = torch.cat([dists, torch.Tensor([1e10]).expand(dists[..., :1].shape).to(rays_o.device)], -1)
    
    alpha_coarse = raw2alpha(sigma_coarse_raw, dists)
    T_coarse = torch.cumprod(1.0 - alpha_coarse + 1e-10, dim=-1) # [R, N]
    T_coarse = torch.cat([torch.ones([R, 1]).to(rays_o.device), T_coarse[..., :-1]], -1)
    weights_coarse = alpha_coarse * T_coarse
    
    rgb_coarse = torch.sum(weights_coarse[..., None] * torch.sigmoid(rgb_coarse_raw), -2)
    acc_coarse = torch.sum(weights_coarse, -1)
    depth_coarse = torch.sum(weights_coarse * z_vals, -1) / acc_coarse
    
    # 3. Fine Sampling
    if N_importance > 0:
        # Resample based on coarse weights
        # Sample from discrete distribution defined by weights_coarse
        # Note: In practice, this often involves sampling indices or using inverse CDF
        # Here we assume a helper `sample_finer_z_values` exists which does this
        finer_z_vals = sample_finer_z_values(z_vals, weights_coarse, N_importance)
        
        # Concatenate and sort
        z_vals_fine = torch.cat([z_vals, finer_z_vals], -1)
        z_vals_fine, _ = torch.sort(z_vals_fine, dim=-1)
        
        # Detach fine samples from graph
        z_vals_fine = z_vals_fine.detach()
        
        # Recompute points and query Fine Model
        pts_fine = rays_o[..., None, :] + rays_d[..., None, :] * z_vals_fine[..., :, None]
        pts_fine_flat = pts_fine.reshape(-1, 3)
        
        if use_viewdirs:
            viewdirs_fine = rays_d[..., None, :].expand(pts_fine.shape)
            viewdirs_fine_flat = viewdirs_fine.reshape(-1, 3)
            pts_fine_enc = positional_encoding(pts_fine_flat, 10)
            viewdirs_fine_enc = positional_encoding(viewdirs_fine_flat, 4)
            raw_fine = fine_model(pts_fine_enc, viewdirs_fine_enc)
            raw_fine = raw_fine.reshape(R, z_vals_fine.shape[1], 4)
        else:
            pts_fine_enc = positional_encoding(pts_fine_flat, 10)
            raw_fine = fine_model(pts_fine_enc)
            raw_fine = raw_fine.reshape(R, z_vals_fine.shape[1], 4)
            
        rgb_fine_raw, sigma_fine_raw = raw_fine[..., :3], raw_fine[..., 3:]
        sigma_fine_raw = sigma_fine_raw + torch.randn_like(sigma_fine_raw) * raw_noise_std
        
        # Recompute dists for fine samples
        dists_fine = z_vals_fine[..., 1:] - z_vals_fine[..., :-1]
        dists_fine = dists_fine * torch.norm(rays_d[..., None, :], dim=-1)
        dists_fine = torch.cat([dists_fine, torch.Tensor([1e10]).expand(dists_fine[..., :1].shape).to(rays_o.device)], -1)
        
        alpha_fine = raw2alpha(sigma_fine_raw, dists_fine)
        T_fine = torch.cumprod(1.0 - alpha_fine + 1e-10, dim=-1)
        T_fine = torch.cat([torch.ones([R, 1]).to(rays_o.device), T_fine[..., :-1]], -1)
        weights_fine = alpha_fine * T_fine
        
        rgb_fine = torch.sum(weights_fine[..., None] * torch.sigmoid(rgb_fine_raw), -2)
        acc_fine = torch.sum(weights_fine, -1)
        depth_fine = torch.sum(weights_fine * z_vals_fine, -1) / acc_fine
    else:
        rgb_fine = rgb_coarse
        acc_fine = acc_coarse
        depth_fine = depth_coarse
        
    # 4. Background Compositing
    if white_bkgd:
        rgb_coarse = rgb_coarse + (1.0 - acc_coarse[..., None])
        rgb_fine = rgb_fine + (1.0 - acc_fine[..., None])
        
    return {
        'rgb_coarse': rgb_coarse,
        'rgb_fine': rgb_fine,
        'acc_coarse': acc_coarse,
        'acc_fine': acc_fine,
        'depth_coarse': depth_coarse,
        'depth_fine': depth_fine
    }

def sample_finer_z_values(z_vals, weights, N_importance):
    """
    Sample N_importance points from the distribution defined by weights.
    This is a placeholder for the actual resampling logic (e.g., inverse CDF sampling).
    """
    # Normalize weights to sum to 1
    weights = weights / (weights.sum(-1, keepdim=True) + 1e-10)
    
    # Cumulative sum
    cdf = weights.cumsum(-1)
    cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], -1)
    
    # Sample uniform random numbers
    u = torch.rand([z_vals.shape[0], N_importance]).to(z_vals.device)
    
    # Inverse CDF
    indices = torch.searchsorted(cdf, u, right=True)
    # Clamp indices
    indices = torch.clamp(indices, 0, z_vals.shape[1] - 1)
    
    # Interpolate or just pick the z_vals at these indices
    # Simple version: pick the z_vals at the sampled indices
    z_fine = torch.gather(z_vals, -1, indices)
    return z_fine
```

### Parameters with Typical Values

*   `N_samples_coarse`: 64 [S8, S19, unofficial].
*   `N_importance`: 128 [S19, unofficial].
*   `perturb`: 1.0 (jitter) or 0.0 (no jitter) [S7, S19, unofficial].
*   `raw_noise_std`: 1.0 (standard deviation of noise added to density) [S9, S19, unofficial].
*   `lindisp`: False (sample in depth) or True (sample in inverse depth) [S8, S19, unofficial].
*   `white_bkgd`: False (transparent/black background) or True (white background) [S7, S19, unofficial].
*   `use_viewdirs`: False (view-independent color) or True (view-dependent color) [S19, unofficial].
*   `multires` (position encoding bands): 10 [S19, unofficial].
*   `multires_views` (direction encoding bands): 4 [S19, unofficial].
*   `netdepth` (coarse MLP layers): 8 [S19, unofficial].
*   `netwidth` (coarse MLP channels): 256 [S19, unofficial].
*   `netdepth_fine` (fine MLP layers): 8 [S19, unofficial].
*   `netwidth_fine` (fine MLP channels): 256 [S19, unofficial].
*   `lrate` (learning rate): $5 \times 10^{-4}$ [S19, unofficial].
*   `lrate_decay`: 250,000 steps [S19, unofficial].

### Edge Cases and Pitfalls

*   **Numerical Stability:** A small constant $\epsilon = 10^{-10}$ is added to the cumulative product of transmittance to prevent underflow [S12, S19, unofficial].
*   **Last Segment:** The last segment distance is set to a large value ($10^{10}$) to ensure the ray terminates and the last sample contributes to the accumulation [S9, unofficial][S11].
*   **Gradient Flow:** Fine samples are detached from the computation graph to prevent gradients from flowing back to the coarse network, ensuring the coarse network only learns to identify opaque regions, not to render the final image [S18, unofficial].
*   **View Direction:** If `use_viewdirs` is False, the network predicts view-independent color, which can lead to artifacts in specular highlights or glossy materials [S19, unofficial].
*   **Near/Far Bounds:** Incorrect near/far bounds can lead to samples being placed outside the scene or missing parts of the scene, resulting in black or incorrect renderings [S5, S8, unofficial].

## Subtopics

### [NeRF Importance Sampling](1565-nerf-importance-sampling.md)
A technique to refine the sampling distribution along rays by using the weights from a coarse model to generate additional sample points, improving rendering quality with minimal overhead. It concentrates computational effort on regions of high opacity by resampling based on coarse alpha compositing weights. This is the core mechanism of the coarse/fine sampling described above, where the fine stage samples from the coarse distribution. See [NeRF Importance Sampling](1565-nerf-importance-sampling.md).

### [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md)
The NerfAcc Rendering Pipeline is a modular, high-performance volumetric rendering method implemented as a PyTorch library. It decouples the sampling strategy from the radiance field representation, supporting both Occupancy Grids and Proposal Networks to accelerate training and inference. Unlike standard uniform sampling, it uses interval-based ray representation to support conical frustums and integrates coarse/fine sampling with importance sampling to concentrate samples on surfaces while skipping empty space. It builds on [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md) by providing a unified API for efficient transmittance estimation and differentiable rendering. See [NerfAcc Rendering Pipeline](1619-nerfacc-rendering-pipeline.md).

### [VF-NeRF](3138-vf-nerf.md)
VF-NeRF is a neural volume rendering method that represents indoor scenes using learned vector fields rather than standard scalar density fields, trained from multi-view posed images and their associated monocular depth maps. It distinguishes itself from standard NeRF by modeling scene geometry and appearance through a vector field representation integrated along rays, leveraging structural priors from depth supervision. See [VF-NeRF](3138-vf-nerf.md).

### Voxel-based Volume Rendering
A volume rendering technique that trains a voxel representation of a radiance field, where multiple voxels contribute to a single sample during rendering [S1, unofficial]. It differs from voxelization (sparse octree) in that voxel-based rendering uses dense or semi-dense voxels where multiple voxels contribute to a single sample, whereas voxelization creates a sparse octree containing voxels only where the model determines the space is solid [S1, unofficial]. It is used as a coarse model for two-tier sampling in NeRF-like architectures to determine opaque regions in space [S1, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(R \cdot (N_{coarse} + N_{fine}))$ per batch of rays, where $R$ is the number of rays, $N_{coarse}$ is the number of coarse samples, and $N_{fine}$ is the number of fine samples. The coarse stage is $O(R \cdot N_{coarse})$ and the fine stage is $O(R \cdot (N_{coarse} + N_{fine}))$.
*   **Memory Complexity:** $O(R \cdot (N_{coarse} + N_{fine}))$ to store intermediate tensors (points, densities, alphas, transmittances, weights) for the batch of rays.
*   **Measured Performance:** Training a NeRF on a single scene (e.g., Lego, Fern) takes several hours to a day on a single GPU, depending on resolution and number of samples [S7, unofficial]. Rendering an image takes less than a second to ~30 seconds, depending on resolution and chunking [S7, unofficial].
*   **Scaling:** Performance scales linearly with the number of rays and samples. Memory usage can be managed by processing rays in chunks (e.g., `chunk_size` = 32768) [S11][S19, unofficial].
*   **Strengths:** High-quality novel view synthesis, differentiable rendering allows optimization from images alone, handles complex geometry and appearance.
*   **Limitations:** Slow training and rendering compared to explicit representations, requires many input views for good results, sensitive to near/far bounds and sampling strategies.
*   **When to Use:** When high-quality novel view synthesis is required and computational resources are available for training.
*   **When Not to Use:** When real-time rendering is required without pre-training, or when input views are sparse or unposed.

## Background

NeRF (Neural Radiance Fields) was introduced by Mildenhall et al. in 2020 [S3][S7, unofficial]. It represents a 3D scene as a continuous volumetric function family that maps 3D coordinates and viewing directions to volume density and RGB color, modeled as an MLP [S7, unofficial][S10, S24]. The method synthesizes novel views by querying this function along camera rays and integrating the outputs using classical volume rendering techniques [S3, S14]. The coarse/fine sampling strategy was introduced to address the issue of low-resolution convergence and inefficient sampling in single-stage neural volume rendering [S8, S13, unofficial]. It has become a foundational method in neural rendering and view synthesis [S3].

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
[S15] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
[S16] Fast Learning Radiance Fields by Shooting Much Fewer Rays, https://doi.org/10.1109/tip.2023.3267049, fetched 2026-10-26
[S17] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
[S18] charan2598/NeRF-PyTorch, https://github.com/charan2598/NeRF-PyTorch, fetched 2026-10-27 (unofficial)
[S19] dgsmith7/nerf-mlp, https://github.com/dgsmith7/nerf-mlp, fetched 2026-10-27 (unofficial)
[S20] 3DGS-101/textbook/05-alpha-compositing.md at main - GitHub, https://github.com/Chamud/3DGS-101/blob/main/textbook/05-alpha-compositing.md, fetched 2026-10-28 (unofficial)
[S21] VoxNeRF: Bridging Voxel Representation and Neural Radiance Fields for ..., https://arxiv.org/abs/2311.05289, fetched 2026-10-29
[S22] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S23] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S24] NeRFMeshing: Distilling Neural Radiance Fields into Geometrically-Accurate 3D Meshes, https://doi.org/10.1109/3dv62453.2024.00093, fetched 2026-10-29
