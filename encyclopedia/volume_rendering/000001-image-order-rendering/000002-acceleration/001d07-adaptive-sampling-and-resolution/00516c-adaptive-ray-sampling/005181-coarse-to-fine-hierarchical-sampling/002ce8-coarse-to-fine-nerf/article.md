# Coarse-to-Fine NeRF

Also called: Coarse-Fine NeRF, Hierarchical Rendering (Original NeRF), Two-Stage NeRF

## Overview
Coarse-to-Fine NeRF is a hierarchical sampling strategy that improves the learning of high-frequency details in Neural Radiance Fields by performing two tiers of volume sampling [S1, S3, unofficial]. It solves the problem of inefficient uniform sampling, where most samples fall in empty space, by using a coarse network to identify opaque regions and then clustering a second set of samples near those regions to train a fine network [S1, S4, unofficial]. It sets itself apart from standard uniform sampling by adaptively concentrating computational effort on visible geometry, and from its sibling [TermiNeRF](8828-terminerf.md) by not explicitly modeling termination probabilities but rather relying on opacity-based importance sampling [S1, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origins $\mathbf{o} \in \mathbb{R}^3$, ray directions $\mathbf{d} \in \mathbb{R}^3$, near bound $t_{near}$, far bound $t_{far}$, number of coarse samples $N_c$, number of fine samples $N_f$.
*   **Outputs:** Rendered RGB color $\mathbf{C}(\mathbf{r})$, depth map, accumulated alpha, and weights for both coarse and fine passes.

**Data Structures:**
*   **Networks:** Two separate Multi-Layer Perceptrons (MLPs), `network_coarse` and `network_fine`. Both share the same architecture (hidden dimensions, layers) but have distinct parameters [S3, unofficial].
*   **Positional Encoding:** Input coordinates are encoded using positional encoding with `pos_levels` (default 10) and view directions with `dir_levels` (default 4) [S3, unofficial].
*   **Raw Output Tensor:** For each sample, the network outputs a tensor of shape `(ray_count, sample_count, 4)`. The first 3 channels are RGB (sigmoided to $[0,1]$), and the 4th channel is the raw volume density $\sigma$ [S5, unofficial].

**Parameters:**
*   `n_coarse`: Number of coarse samples per ray. Default: 64 [S4, unofficial].
*   `n_fine`: Number of fine samples per ray. Default: 128 [S4, unofficial].
*   `perturb`: Boolean flag to enable stratified jittering during coarse sampling. Default: True [S4, unofficial].
*   `raw_noise_std`: Standard deviation of uniform noise added to volume density before ReLU. Default: 0 [S5, unofficial].
*   `white_bkgd`: Boolean flag. If True, adds unaccounted transmittance to RGB assuming a white background [S5, unofficial].

**Steps:**

1.  **Coarse Sampling:** Generate $N_c$ depth values $t_c$ uniformly spaced between $t_{near}$ and $t_{far}$. If `perturb` is True, apply stratified sampling by adding uniform noise $\mathcal{U}[0, 1]$ to break symmetry and reduce aliasing [S4, S5, unofficial].
2.  **Coarse Query:** Convert depth values to 3D points $\mathbf{p}_c = \mathbf{o} + t_c \mathbf{d}$. Query `network_coarse` with $\mathbf{p}_c$ and encoded directions to obtain raw outputs $\mathbf{raw}_c$ [S4, S5, unofficial].
3.  **Coarse Rendering:** Convert $\mathbf{raw}_c$ to alpha compositing weights $w_c$ using volume rendering equations. Compute the coarse RGB color $\mathbf{C}_c$ and depth [S5, unofficial].
4.  **Hierarchical Sampling:** Use the coarse weights $w_c$ to define a probability distribution. Sample $N_f$ new depth values $t_f$ from this distribution using inverse transform sampling (or `sample_pdf`) [S4, S5, unofficial].
    *   *Pitfall:* The sampling step is non-differentiable. Gradients must not flow back through this step to the coarse network. The coarse samples/weights used for fine sampling must be detached from the computation graph [S4, S5, unofficial].
5.  **Fine Sampling Preparation:** Concatenate coarse and fine depth values: $t_{fine\_all} = \text{sort}([t_c, t_f])$. Convert these to 3D points $\mathbf{p}_{fine\_all}$ [S4, S5, unofficial].
6.  **Fine Query:** Query `network_fine` with $\mathbf{p}_{fine\_all}$ and encoded directions to obtain $\mathbf{raw}_f$ [S5, unofficial].
7.  **Fine Rendering:** Convert $\mathbf{raw}_f$ to weights $w_f$ and compute the final RGB color $\mathbf{C}_f$ [S5, unofficial].

**Formulas:**

*   **Distance between samples:**
    $$ \delta_i = z_{i+1} - z_i $$
    An infinite distance ($10^{10}$) is appended to the last column to ensure the opacity of the last segment is 1 [S5, unofficial].

*   **Alpha (Opacity):**
    $$ \alpha = 1 - \exp(-\sigma \cdot \delta) $$
    Where $\sigma$ is the volume density from the network output, optionally perturbed by noise $\mathcal{U}[0,1]$ if `raw_noise_std` > 0, and passed through ReLU [S5, unofficial].

*   **Transmittance:**
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
    Computed as the cumulative product of $(1 - \alpha)$, prepended with 1. The last column is discarded [S5, unofficial].

*   **Weights:**
    $$ w_i = \alpha_i \cdot T_i $$

*   **RGB Rendering:**
    $$ \mathbf{C} = \sum_{i=1}^{N} w_i \mathbf{c}_i $$
    If `white_bkgd` is True: $\mathbf{C} = \mathbf{C} + (1 - \sum w_i) \cdot \mathbf{1}$ [S5, unofficial].

*   **Loss Function:**
    $$ L = L_{coarse} + L_{fine} $$
    $$ L_{coarse} = \text{mean}((\mathbf{C}_{coarse} - \mathbf{C}_{target})^2) $$
    $$ L_{fine} = \text{mean}((\mathbf{C}_{fine} - \mathbf{C}_{target})^2) $$
    Both networks are supervised against the target pixel color [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def raw2outputs(raw, z_vals, rays_d, raw_noise_std=0, white_bkgd=False):
    """
    Converts raw network outputs to radiance and weights.
    raw: (ray_count, sample_count, 4) -> RGB (0-1) and sigma
    z_vals: (ray_count, sample_count)
    rays_d: (ray_count, 3)
    """
    # Calculate distances between samples
    dists = z_vals[..., 1:] - z_vals[..., :-1]
    # Append infinite distance for the last segment
    dists = torch.cat([dists, torch.full_like(dists[..., :1], 1e10)], dim=-1)
    # Scale by ray direction norm
    dists = dists * torch.norm(rays_d[..., None, :], dim=-1)
    
    # Extract RGB and Sigma
    rgb = torch.sigmoid(raw[..., :3])
    sigma_raw = raw[..., 3]
    
    # Add noise to sigma if specified
    noise = 0.
    if raw_noise_std > 0.:
        noise = torch.randn_like(sigma_raw) * raw_noise_std
        
    # Apply ReLU and compute alpha
    sigma = F.relu(sigma_raw + noise)
    alpha = 1. - torch.exp(-sigma * dists)
    
    # Compute transmittance T
    # T_i = prod_{j<i} (1 - alpha_j)
    transmittance_prev = torch.cat([torch.ones_like(alpha[..., :1]), 
                                    1. - alpha + 1e-10], dim=-1)
    T = torch.cumprod(transmittance_prev, dim=-1)[..., :-1]
    
    # Compute weights
    weights = alpha * T
    
    # Render RGB
    rgb_map = torch.sum(weights[..., None] * rgb, dim=-2)
    
    # Render Depth
    depth_map = torch.sum(weights * z_vals, dim=-1)
    
    # Accumulated Alpha
    acc_map = torch.sum(weights, dim=-1)
    
    # White background
    if white_bkgd:
        rgb_map = rgb_map + (1. - acc_map[..., None])
        
    return rgb_map, depth_map, acc_map, weights

def stratified_sample(near, far, N_c, ray_o, ray_d, perturb=True):
    """
    Uniform stratified sampling along rays.
    Returns: pts (B, N_c, 3), dirs (B, N_c, 3), t_vals (B, N_c)
    """
    device = ray_o.device
    t_vals = torch.linspace(near, far, N_c, device=device)
    t_vals = t_vals.expand(*ray_o.shape[:-1], N_c)
    
    if perturb:
        mid = (t_vals[..., :-1] + t_vals[..., 1:]) / 2.0
        upper = torch.cat([mid, t_vals[..., -1:]], dim=-1)
        lower = torch.cat([t_vals[..., :1], mid], dim=-1)
        t_vals = lower + (upper - lower) * torch.rand_like(t_vals)
        
    pts = ray_o.unsqueeze(-2) + ray_d.unsqueeze(-2) * t_vals.unsqueeze(-1)
    dirs = ray_d.unsqueeze(-2).expand_as(pts)
    return pts, dirs, t_vals

def sample_pdf(bins, weights, N_f, perturb=True):
    """
    Hierarchical sampling based on coarse weights.
    bins: (B, N_c) sorted depth values from coarse pass
    weights: (B, N_c) alpha compositing weights from coarse pass
    Returns: t_fine (B, N_f)
    """
    # Get midpoints of coarse bins for PDF definition
    # weights[i] corresponds to the segment between bins[i] and bins[i+1]
    # We need to handle the last segment or use the provided bins as boundaries
    
    # Calculate PDF from weights
    # weights shape: (B, N_c). bins shape: (B, N_c)
    # The weight w_i is for the interval [bins[i], bins[i+1]]? 
    # In standard NeRF implementation, weights are computed for samples at bins.
    # We construct a CDF from weights and sample from it.
    
    # Normalize weights to sum to 1 per ray
    weights = weights + 1e-5 # prevent NaN
    pdf = weights / torch.sum(weights, dim=-1, keepdim=True)
    cdf = torch.cumsum(pdf, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[..., :1]), cdf], dim=-1)
    
    # Sample uniform values
    u = torch.rand(*cdf.shape[:-1], N_f, device=weights.device)
    
    # Inverse transform sampling
    # Find indices where cdf < u
    indices = torch.searchsorted(cdf[..., :-1], u, right=True)
    # Clamp indices
    indices = torch.clamp(indices, 0, cdf.shape[-2] - 2)
    
    # Linear interpolation within bins
    g = indices.float()
    bin_low = bins[torch.arange(cdf.shape[0]).unsqueeze(-1), indices]
    bin_high = bins[torch.arange(cdf.shape[0]).unsqueeze(-1), indices + 1]
    
    # Normalize u to the bin's CDF range
    cdf_low = cdf[torch.arange(cdf.shape[0]).unsqueeze(-1), indices]
    cdf_high = cdf[torch.arange(cdf.shape[0]).unsqueeze(-1), indices + 1]
    
    t = bin_low + (bin_high - bin_low) * ((u - cdf_low) / (cdf_high - cdf_low + 1e-5))
    
    if perturb:
        # Add jitter
        dists = bin_high - bin_low
        t = t + torch.rand_like(t) * dists
        
    return t

def render_rays(rays_o, rays_d, network_coarse, network_fine, 
                near, far, n_coarse=64, n_fine=128, 
                raw_noise_std=0, white_bkgd=False, perturb=True):
    """
    Full coarse-to-fine rendering pipeline.
    """
    # 1. Coarse Sampling
    pts_coarse, dirs_coarse, t_coarse = stratified_sample(
        near, far, n_coarse, rays_o, rays_d, perturb
    )
    
    # 2. Coarse Query
    # Assume network takes (pts, dirs) and returns raw (B, N, 4)
    raw_coarse = network_coarse(pts_coarse, dirs_coarse)
    
    # 3. Coarse Rendering
    rgb_coarse, depth_coarse, acc_coarse, weights_coarse = raw2outputs(
        raw_coarse, t_coarse, rays_d, raw_noise_std, white_bkgd
    )
    
    # 4. Hierarchical Sampling
    # Detach to prevent gradient flow from fine to coarse sampling logic
    weights_coarse_detached = weights_coarse.detach()
    t_coarse_detached = t_coarse.detach()
    
    t_fine = sample_pdf(t_coarse_detached, weights_coarse_detached, n_fine, perturb)
    
    # 5. Fine Sampling Preparation
    t_fine_all = torch.cat([t_coarse_detached, t_fine], dim=-1)
    t_fine_all, _ = torch.sort(t_fine_all, dim=-1)
    
    pts_fine_all = rays_o.unsqueeze(-2) + rays_d.unsqueeze(-2) * t_fine_all.unsqueeze(-1)
    dirs_fine_all = rays_d.unsqueeze(-2).expand_as(pts_fine_all)
    
    # 6. Fine Query
    raw_fine = network_fine(pts_fine_all, dirs_fine_all)
    
    # 7. Fine Rendering
    rgb_fine, depth_fine, acc_fine, weights_fine = raw2outputs(
        raw_fine, t_fine_all, rays_d, raw_noise_std, white_bkgd
    )
    
    return {
        'rgb_coarse': rgb_coarse,
        'depth_coarse': depth_coarse,
        'rgb_fine': rgb_fine,
        'depth_fine': depth_fine,
        'weights_coarse': weights_coarse,
        'weights_fine': weights_fine
    }
```

## Subtopics

### Single Multiscale MLP Architecture
A variant that uses a single MLP with parameters $\Theta$ for both coarse and fine stages, enabled by Integrated Positional Encoding (IPE) which explicitly encodes scale into input features [S6]. Unlike standard NeRF which requires two distinct MLPs because standard PE only models one scale, this approach allows one MLP to learn a multiscale representation [S6]. The optimization balances coarse and fine losses using a hyperparameter $\lambda$ (default 0.1) [S6].

### Segmented Sampling
A strategy that selectively increases the sampling rate only within the bounding box of an object area to ensure detailed sampling without significantly increasing overall computational cost [S7]. It addresses the pitfall where standard uniform sampling results in sparse points on small objects relative to the scene, causing missing areas in rendered images due to inaccurate PDF in the fine stage [S7].

### DONeRF PDF Sampling
A method used in DONeRF to translate depth oracle classification outputs into sample locations. It treats the classified depth output vector from a single oracle network evaluation as a piecewise-constant Probability Density Function (PDF) and samples from its Cumulative Distribution Function (CDF) [S8]. Unlike standard NeRF which builds a PDF from opacity outputs of a coarse shading network evaluated at multiple locations, this uses the oracle's direct output [S8].

## Operating characteristics
*   **Time Complexity:** $O(N_c + N_f)$ network queries per ray, where $N_c$ is coarse samples and $N_f$ is fine samples. Typically $N_c=64, N_f=128$ [S4, unofficial].
*   **Memory Complexity:** Stores outputs for both coarse and fine passes. The compute graph must be static; the coarse-to-fine edge is cut via `.detach()` to prevent cyclic definitions [S5, unofficial].
*   **Performance:** Improves detail learning by clustering samples near opaque regions, unlike evenly distributed sampling [S1, unofficial]. It prevents dense sampling at fine scales in empty space, improving computational efficiency compared to uniformly dense sampling [S2].
*   **Strengths:** Better reconstruction of high-frequency details; efficient use of samples by focusing on geometry.
*   **Limitations:** Requires two network passes (or two networks); non-differentiable sampling step breaks gradient flow from fine to coarse [S4, S5, unofficial].

## Background
Coarse-to-Fine NeRF, also known as Hierarchical Rendering, was introduced in the original NeRF paper by Mildenhall et al. [S2][S4, unofficial]. It was designed to improve computational efficiency and rendering quality by using a coarse network to guide the sampling of a fine network [S2]. The method has become a standard component in many subsequent NeRF variants [S2].

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S5] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S6] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S7] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
[S8] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
