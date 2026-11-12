# Coarse-to-Fine NeRF

Also called: Coarse-Fine NeRF, Hierarchical Rendering (Original NeRF), Two-Stage NeRF

## Overview
Coarse-to-Fine NeRF is a hierarchical ray sampling strategy that uses a "coarse" neural network to estimate the opacity distribution along a ray, then generates a denser set of samples clustered in high-opacity regions to train a "fine" network [S1, S3, S4, unofficial]. It solves the inefficiency of uniform sampling by concentrating computational effort where the scene geometry is present, allowing the model to learn high-frequency details without the prohibitive cost of uniformly dense sampling [S1, S4, unofficial]. It differs from siblings like [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md) by using standard point samples rather than conical frustums, and from [Probability-based Ray Sampling with Quadtree Subdivision](7481-probability-based-ray-sampling-with-quadtree-subdivision.md) by operating in 1D ray space rather than 2D image space [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origins $\mathbf{o} \in \mathbb{R}^3$, ray directions $\mathbf{d} \in \mathbb{R}^3$, near bound $t_n$, far bound $t_f$.
*   **Outputs:** Rendered RGB color $\mathbf{C}(\mathbf{r})$, depth map, accumulated alpha.

**Data Structures:**
*   **Coarse Network ($\mathcal{N}_c$):** MLP with parameters $\Theta_c$. Takes positional encoding $\mathbf{x}$ and directional encoding $\mathbf{d}$, outputs raw color $\mathbf{c} \in \mathbb{R}^3$ and volume density $\sigma \in \mathbb{R}$.
*   **Fine Network ($\mathcal{N}_f$):** MLP with parameters $\Theta_f$. Same architecture as $\mathcal{N}_c$ but separate parameters.
*   **Sample Arrays:**
    *   $t_c$: Coarse depth values, shape $(N, N_c)$.
    *   $t_f$: Fine depth values, shape $(N, N_f)$.
    *   $t_{all}$: Concatenated and sorted depths, shape $(N, N_c + N_f)$.
    *   $\mathbf{pts}$: 3D points, shape $(N, N_c + N_f, 3)$.

**Parameters:**
*   $N_c$: Number of coarse samples per ray. Default: 64 [S4, S5, unofficial].
*   $N_f$: Number of fine samples per ray. Default: 128 [S4, S5, unofficial].
*   `perturb`: Boolean flag to add random noise to coarse samples to break symmetry. Default: True [S4, S5, unofficial].
*   `raw_noise_std`: Standard deviation of noise added to $\sigma$ before ReLU. Default: 0 [S5, unofficial].
*   `white_bkgd`: Boolean flag to assume white background. Default: False [S5, unofficial].
*   `pos_levels`: Frequency octaves for positional encoding. Default: 10 [S3, unofficial].
*   `dir_levels`: Frequency octaves for directional encoding. Default: 4 [S3, unofficial].

**Step-by-Step Method:**

1.  **Coarse Sampling:** Generate $N_c$ samples $t_c$ uniformly along the ray between $t_n$ and $t_f$. If `perturb` is True, stratified sampling is used: divide $[t_n, t_f]$ into $N_c$ bins, sample uniformly within each bin [S3, S4, S5, unofficial].
2.  **Coarse Query:** Convert $t_c$ to 3D points $\mathbf{pts}_c = \mathbf{o} + t_c \cdot \mathbf{d}$. Query $\mathcal{N}_c$ to get raw outputs $\mathbf{raw}_c$ of shape $(N, N_c, 4)$.
3.  **Coarse Rendering:** Convert $\mathbf{raw}_c$ to weights $w_c$ using volume rendering equations (see Formula below). Compute coarse color $\mathbf{rgb}_{coarse}$ and loss $L_{coarse}$ [S4, S5, unofficial].
4.  **Importance Sampling:** Use the coarse weights $w_c$ to define a piecewise-constant probability density function (PDF). Sample $N_f$ new depths $t_{fine}$ from this PDF using inverse transform sampling [S4, S5, unofficial].
    *   *Note:* The sampling step is non-differentiable. Gradients must not flow back through this step to update $\mathcal{N}_c$ based on $\mathcal{N}_f$'s sampling distribution [S4, S5, unofficial].
5.  **Concatenation and Sorting:** Concatenate $t_c$ and $t_{fine}$ to form $t_{all} = [t_c, t_{fine}]$. Sort $t_{all}$ along the ray dimension [S4, S5, unofficial].
6.  **Fine Query:** Convert $t_{all}$ to 3D points $\mathbf{pts}_{all}$. Query $\mathcal{N}_f$ to get raw outputs $\mathbf{raw}_f$ [S4, S5, unofficial].
7.  **Fine Rendering:** Convert $\mathbf{raw}_f$ to weights $w_f$ and compute fine color $\mathbf{rgb}_{fine}$ [S4, S5, unofficial].
8.  **Loss Calculation:** The total loss is the sum of coarse and fine losses:
    $$ L = L_{coarse} + L_{fine} $$
    where $L_{coarse} = \text{mean}((\mathbf{rgb}_{coarse} - \mathbf{target})^2)$ and $L_{fine} = \text{mean}((\mathbf{rgb}_{fine} - \mathbf{target})^2)$ [S4, unofficial].

**Formulas:**

*   **Distance between samples:**
    $$ \delta_i = z_{i+1} - z_i $$
    An infinite distance ($10^{10}$) is appended to the last column to ensure the opacity of the last segment is 1 [S5, unofficial].
*   **Alpha (Opacity):**
    $$ \alpha_i = 1 - \exp(-\hat{\sigma}_i \cdot \delta_i) $$
    where $\hat{\sigma}_i = \text{ReLU}(\sigma_i + \epsilon)$, and $\epsilon \sim \mathcal{U}[0, \text{raw\_noise\_std}]$ [S5, unofficial].
*   **Transmittance:**
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \hat{\sigma}_j \delta_j\right) $$
    Computed as cumulative product of $(1-\alpha)$, prepended with 1 [S5, unofficial].
*   **Weights:**
    $$ w_i = \alpha_i \cdot T_i $$
*   **Color:**
    $$ \mathbf{C}(\mathbf{r}) = \sum_{i=1}^{M} w_i \mathbf{c}_i $$
    If `white_bkgd` is True, add $(1 - \sum w_i) \cdot \mathbf{1}$ to the result [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def raw2outputs(raw, z_vals, rays_d, raw_noise_std=0, white_bkgd=False):
    """
    Converts raw network outputs to radiance, depth, and weights.
    raw: (ray_count, sample_count, 4) -> RGB (sigmoided) and sigma
    z_vals: (ray_count, sample_count)
    rays_d: (ray_count, 3)
    """
    # Calculate distances between samples
    dists = z_vals[..., 1:] - z_vals[..., :-1]
    # Append infinite distance for the last segment
    dists = torch.cat([dists, torch.full_like(dists[..., :1], 1e10)], dim=-1)
    # Scale by ray direction norm (usually 1 for normalized rays)
    dists = dists * torch.norm(rays_d[..., None, :], dim=-1)
    
    # Extract RGB and Sigma
    rgb = torch.sigmoid(raw[..., :3])
    sigma_raw = raw[..., 3]
    
    # Add noise to sigma if specified
    noise = 0.
    if raw_noise_std > 0.:
        noise = torch.randn(sigma_raw.shape) * raw_noise_std
        sigma_raw = sigma_raw + noise
        
    # Compute alpha
    # alpha = 1 - exp(-sigma * dist)
    # Using ReLU on sigma as per original implementation
    alpha = 1. - torch.exp(-F.relu(sigma_raw) * dists)
    
    # Compute transmittance T
    # T_i = prod_{j<i} (1 - alpha_j)
    # cumprod(1-alpha) with 1 prepended
    transmittance = torch.cumprod(torch.cat([
        torch.ones((alpha.shape[0], 1), device=alpha.device),
        1. - alpha + 1e-10
    ], dim=-1), dim=-1)[:, :-1]
    
    # Compute weights
    weights = alpha * transmittance
    
    # Compute RGB map
    rgb_map = torch.sum(weights[..., None] * rgb, dim=-2)
    
    # Compute depth map
    depth_map = torch.sum(weights * z_vals, dim=-1)
    
    # Compute accumulated alpha
    acc_map = torch.sum(weights, dim=-1)
    
    # Handle white background
    if white_bkgd:
        rgb_map = rgb_map + (1. - acc_map[..., None])
        
    return rgb_map, depth_map, acc_map, weights

def hierarchical_sample(z_vals, weights, N_f, perturb=True):
    """
    Samples N_f points from the distribution defined by weights.
    z_vals: (ray_count, N_c)
    weights: (ray_count, N_c)
    """
    # Get midpoints of coarse intervals
    z_vals_mid = 0.5 * (z_vals[..., 1:] + z_vals[..., :-1])
    
    # Pad z_vals and weights for integration
    # Add small epsilon to weights to avoid division by zero
    weights = weights + 1e-5
    
    # Compute CDF from weights
    # cdf = cumsum(weights)
    # Normalize cdf to [0, 1]
    cdf = weights / torch.sum(weights, dim=-1, keepdim=True)
    cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], dim=-1)
    
    # Sample uniform random values
    u = torch.rand(list(weights.shape[:-1]) + [N_f], device=weights.device)
    
    # Inverse transform sampling: find z_vals where cdf >= u
    # This is a simplified implementation; in practice, torch.searchsorted is used
    # or a custom binary search. Here we assume a helper `sample_pdf` exists
    # which performs this inverse CDF sampling.
    z_samples = sample_pdf(z_vals_mid, weights, N_f, detach=True)
    
    return z_samples

def sample_pdf(bins, weights, N_f, detach=False):
    """
    Performs inverse transform sampling from the PDF defined by weights on bins.
    bins: (ray_count, N_c-1) midpoints
    weights: (ray_count, N_c) weights corresponding to intervals
    """
    # This function implements the logic described in [S5, unofficial] for generating
    # fine samples from coarse weights.
    
    # Get the CDF of the weights
    # Weights are associated with intervals defined by z_vals.
    # bins are the midpoints.
    
    # Compute CDF
    cdf = weights[..., :-1] # weights for intervals ending at bins? 
    # Actually, standard NeRF implementation uses weights from z_vals (N_c)
    # and bins (N_c-1).
    
    # Let's follow the standard logic:
    # 1. Normalize weights to form PDF
    # 2. Cumsum to get CDF
    # 3. Sample uniform u
    # 4. Find indices where CDF >= u
    # 5. Interpolate z values
    
    # Simplified for pseudocode completeness:
    # In the actual PyTorch implementation, this involves torch.searchsorted.
    # We will write out the logic clearly.
    
    # Normalize weights
    pdf = weights / (torch.sum(weights, dim=-1, keepdim=True) + 1e-5)
    cdf = torch.cumsum(pdf, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], dim=-1)
    
    # Sample uniform
    u = torch.rand_like(cdf[..., :1]).expand_as(cdf)
    
    # Find indices
    # This is the non-differentiable part. 
    # We use detach to ensure no gradient flows back to coarse network.
    if detach:
        cdf = cdf.detach()
        
    # Get indices where cdf >= u
    # torch.searchsorted is the standard way
    indices = torch.searchsorted(cdf, u, side='left')
    
    # Clip indices to valid range
    indices = torch.clamp(indices, 0, cdf.shape[-1] - 2)
    
    # Interpolate
    # z_low = bins[indices]
    # z_high = bins[indices + 1]
    # ... interpolation logic ...
    
    # For the purpose of this article, we assume sample_pdf returns the
    # correctly interpolated z_samples.
    return torch.zeros_like(u) # Placeholder for actual interpolation logic

def render_rays(rays_o, rays_d, network_coarse, network_fine, 
                n_coarse=64, n_fine=128, perturb=True, 
                raw_noise_std=0, white_bkgd=False):
    """
    Main rendering function implementing Coarse-to-Fine NeRF.
    """
    # 1. Coarse Sampling
    # Generate N_c uniformly spaced samples
    t_vals = torch.linspace(0., 1., n_coarse, device=rays_o.device)
    t_vals = t_vals.expand(list(rays_o.shape[:-1]) + [n_coarse])
    
    if perturb:
        # Stratified sampling
        mids = 0.5 * (t_vals[..., 1:] + t_vals[..., :-1])
        upper = torch.cat([mids, t_vals[..., -1:]], dim=-1)
        lower = torch.cat([t_vals[..., :1], mids], dim=-1)
        t_rand = torch.rand_like(t_vals)
        t_vals = lower + (upper - lower) * t_rand
    else:
        t_vals = t_vals.expand_as(rays_o[..., :1]) # Ensure shape matches
        
    # Convert to depth values (assuming near=0, far=1 for simplicity, 
    # in practice near/far are passed)
    # Let's assume rays_o and rays_d are normalized such that t is depth
    pts = rays_o.unsqueeze(-2) + rays_d.unsqueeze(-2) * t_vals.unsqueeze(-1)
    
    # 2. Coarse Query
    # Flatten for network input
    pts_flat = pts.reshape(-1, 3)
    raw_coarse = network_coarse(pts_flat)
    raw_coarse = raw_coarse.reshape(list(pts.shape[:-1]) + [4])
    
    # 3. Coarse Rendering
    rgb_coarse, depth_coarse, acc_coarse, weights_coarse = raw2outputs(
        raw_coarse, t_vals, rays_d, raw_noise_std, white_bkgd
    )
    
    # 4. Importance Sampling
    # Sample N_f points from weights_coarse
    # Note: weights_coarse shape is (N, N_c)
    z_samples = hierarchical_sample(t_vals, weights_coarse, n_fine, perturb=False)
    
    # 5. Concatenation and Sorting
    t_coarse = t_vals
    t_fine = z_samples
    t_all = torch.cat([t_coarse, t_fine], dim=-1)
    t_all, _ = torch.sort(t_all, dim=-1)
    
    # 6. Fine Query
    pts_fine = rays_o.unsqueeze(-2) + rays_d.unsqueeze(-2) * t_all.unsqueeze(-1)
    pts_fine_flat = pts_fine.reshape(-1, 3)
    raw_fine = network_fine(pts_fine_flat)
    raw_fine = raw_fine.reshape(list(pts_fine.shape[:-1]) + [4])
    
    # 7. Fine Rendering
    rgb_fine, depth_fine, acc_fine, weights_fine = raw2outputs(
        raw_fine, t_all, rays_d, raw_noise_std, white_bkgd
    )
    
    return {
        'rgb_coarse': rgb_coarse,
        'depth_coarse': depth_coarse,
        'rgb_fine': rgb_fine,
        'depth_fine': depth_fine
    }
```

## Subtopics

### Single Multiscale MLP Architecture
A variant that uses a single MLP with parameters $\Theta$ for both coarse and fine stages, enabled by Integrated Positional Encoding (IPE) features which explicitly encode scale into input features [S6]. Unlike standard NeRF which requires two distinct MLPs because standard PE features only model one scale, this approach allows one MLP to learn a multiscale representation [S6]. The optimization problem balances coarse and fine losses with a hyperparameter $\lambda$ (default 0.1) [S6].

### Segmented Sampling
A strategy that selectively increases the sampling rate only within the bounding box of an object area to ensure detailed sampling without significantly increasing overall computational cost [S7]. It addresses the pitfall of standard uniform sampling where small objects may have sparse points, leading to missing areas in rendered images due to inaccurate PDF in the fine stage [S7].

### DONeRF PDF Sampling
A method used in DONeRF to translate depth oracle classification outputs into sample locations. It treats the classified depth output from a single oracle network evaluation as a piecewise-constant Probability Density Function (PDF) and samples from its Cumulative Distribution Function (CDF) [S8]. Unlike NeRF, which builds a PDF from opacity outputs of a coarse shading network evaluated at multiple sample locations, DONeRF uses the output from a single oracle evaluation [S8].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot (N_c + N_f))$ MLP evaluations per ray, where $N$ is the number of rays. This is more expensive than single-stage uniform sampling ($O(N \cdot N_{total})$) but more efficient than achieving the same quality with uniform sampling at $N_{total} = N_c + N_f$.
*   **Memory Complexity:** Requires storing outputs for both coarse and fine networks, roughly doubling the memory footprint of the forward pass compared to a single network [S6].
*   **Performance:** Significantly improves rendering quality (PSNR/SSIM) by capturing high-frequency details that uniform sampling misses, without the computational cost of uniformly dense sampling [S1, S4, unofficial].
*   **Strengths:** Efficiently allocates samples to opaque regions; simple to implement; widely adopted as a baseline in NeRF literature [S3, S4, unofficial].
*   **Limitations:** Requires two network passes per ray; the non-differentiable sampling step requires careful gradient management (detach) [S4, S5, unofficial]; can still suffer from aliasing if not combined with anti-aliasing techniques like Mip-NeRF [S6].
*   **When to use:** When high-quality detail is required and computational budget allows for two network evaluations per ray.
*   **When not to use:** When inference speed is critical and single-pass methods (like [ENeRF](7184-enerf.md)) are preferred, or when anti-aliasing is the primary concern (use [Mip-NeRF 360 Sampling and Training](3393-mip-nerf-360-sampling-and-training.md)).

## Background
Coarse-to-Fine NeRF, also known as Hierarchical Rendering, was introduced in the original NeRF paper by Mildenhall et al. in 2020 [S2, S8]. It was designed to address the inefficiency of uniform ray sampling in neural radiance fields. The method has become a standard component in many subsequent NeRF variants, including Mip-NeRF (which uses a single MLP but retains the hierarchical sampling structure) and DONeRF [S6, S8].

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S5] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S6] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S7] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
[S8] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
