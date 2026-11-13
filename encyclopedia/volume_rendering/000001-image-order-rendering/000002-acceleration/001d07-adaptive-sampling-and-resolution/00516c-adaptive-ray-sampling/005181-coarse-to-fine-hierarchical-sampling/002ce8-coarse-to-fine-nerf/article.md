# Coarse-to-Fine NeRF

Also called: Coarse-Fine NeRF, Hierarchical Rendering (Original NeRF), Two-Stage NeRF

## Overview
Coarse-to-Fine NeRF is a hierarchical sampling strategy that improves the detail and accuracy of neural radiance field (NeRF) rendering by using two distinct networks: a coarse network and a fine network [S1, unofficial][S2][S3, unofficial]. It solves the problem of inefficient uniform sampling, where many samples fall in empty space, by first estimating the opacity distribution of the scene with a coarse pass and then concentrating fine samples in regions of high opacity [S1, S4, unofficial]. Unlike standard uniform sampling, this two-tier approach clusters additional samples near opaque locations, allowing the model to learn high-frequency details without the computational cost of uniformly dense sampling across the entire ray [S1, S4, unofficial]. It sets itself apart from its siblings by establishing the foundational two-stage pipeline where the coarse network's output directly guides the probability density function (PDF) for the fine network's sampling distribution [S2][S4, unofficial].

## How it works
**Inputs and Outputs:**
The method processes rays defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Coarse Network ($\text{MLP}_{coarse}$):** Takes 3D positions and view directions, outputs raw color and volume density.
*   **Fine Network ($\text{MLP}_{fine}$):** Takes 3D positions and view directions, outputs raw color and volume density.
*   **Output:** Final rendered color $\mathbf{C}(\mathbf{r})$, depth, and accumulation maps for both coarse and fine passes.

**Data Structures:**
*   **Ray Parameters:** Near bound $t_n$, far bound $t_f$.
*   **Sample Depths:** 1D tensors $t$ of size $N$ (coarse) or $N_F$ (fine).
*   **Raw Outputs:** Tensor of shape $(N, 4)$, where the first 3 channels are RGB (sigmoided) and the 4th channel is volume density $\sigma$ [S5, unofficial].
*   **Weights:** Tensor of shape $(N,)$ representing the contribution of each sample to the final color [S5, unofficial].

**Parameters:**
*   $N_C$: Number of coarse samples per ray. Default: 64 [S4, unofficial].
*   $N_F$: Number of fine samples per ray. Default: 128 [S4, unofficial].
*   `perturb`: Boolean flag to enable jittering of coarse samples. Default: True [S4, unofficial].
*   `raw_noise_std`: Standard deviation of noise added to $\sigma$ before ReLU. Default: 0 [S5, unofficial].
*   `white_bkgd`: Boolean flag to assume a white background. Default: False [S5, unofficial].
*   `pos_levels`: Frequency octaves for positional encoding. Default: 10 [S3, unofficial].
*   `dir_levels`: Frequency octaves for directional encoding. Default: 4 [S3, unofficial].

**Steps:**

1.  **Coarse Sampling:** Generate $N_C$ sample depths $t_c$ uniformly along the ray between $t_n$ and $t_f$. If `perturb` is True, add uniform noise $\mathcal{U}[0, 1]$ to break symmetry and reduce aliasing [S4, unofficial].
    $$ t_c^{(i)} = t_n + (t_f - t_n) \frac{i + \mathcal{U}[0,1]}{N_C}, \quad i = 0, \dots, N_C-1 $$
2.  **Coarse Network Query:** Query $\text{MLP}_{coarse}$ at positions $\mathbf{x}_c = \mathbf{o} + t_c \mathbf{d}$ to obtain raw outputs $\mathbf{raw}_c \in \mathbb{R}^{N_C \times 4}$ [S4, unofficial].
3.  **Coarse Volume Rendering:** Convert $\mathbf{raw}_c$ to weights $w_c$ using alpha compositing [S5, unofficial]:
    *   Calculate segment distances $\delta_i = t_{c}^{(i+1)} - t_{c}^{(i)}$. Append $\delta_{N_C} = 10^{10}$ to the end [S5, unofficial].
    *   Apply ReLU to density: $\hat{\sigma}_i = \text{ReLU}(\sigma_i + \mathcal{N}(0, \text{raw\_noise\_std}^2))$ [S5, unofficial].
    *   Compute opacity: $\alpha_i = 1 - \exp(-\hat{\sigma}_i \delta_i)$ [S5, unofficial].
    *   Compute transmittance: $T_i = \exp(-\sum_{j=1}^{i-1} \hat{\sigma}_j \delta_j)$, with $T_1 = 1$ [S5, unofficial].
    *   Compute weights: $w_i = \alpha_i T_i$ [S5, unofficial].
4.  **Hierarchical Sampling (Fine):** Use $w_c$ as a PDF to sample $N_F$ new depths $t_f$ via inverse transform sampling [S4, S5, unofficial].
    *   Compute midpoints $t_{mid}^{(i)} = (t_c^{(i)} + t_c^{(i+1)}) / 2$ [S5, unofficial].
    *   Sample $t_f$ from the distribution defined by $w_c$ and $t_{mid}$ [S5, unofficial].
    *   **Crucial:** Detach $t_c$ and $w_c$ from the computation graph to prevent gradients from flowing back through the non-differentiable sampling step to the coarse network [S4, S5, unofficial].
5.  **Fine Network Query:** Concatenate coarse and fine depths $t_{fine} = \text{sort}([t_c, t_f])$. Query $\text{MLP}_{fine}$ at $\mathbf{x}_{fine} = \mathbf{o} + t_{fine} \mathbf{d}$ to obtain $\mathbf{raw}_{fine}$ [S4, S5, unofficial].
6.  **Fine Volume Rendering:** Convert $\mathbf{raw}_{fine}$ to final color $\mathbf{C}_{fine}$ using the same alpha compositing logic as Step 3 [S5, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def raw2outputs(raw, z_vals, rays_d, raw_noise_std=0, white_bkgd=False):
    """
    Converts raw network outputs to volume rendering components.
    raw: (ray_count, sample_count, 4) -> RGB (sigmoided) + sigma
    z_vals: (ray_count, sample_count)
    rays_d: (ray_count, 3)
    """
    # Calculate distances between samples
    dists = z_vals[..., 1:] - z_vals[..., :-1]
    # Append infinite distance for the last segment to force opacity 1
    dists = torch.cat([dists, torch.full_like(dists[..., :1], 1e10)], dim=-1)
    # Scale by ray direction norm
    dists = dists * torch.norm(rays_d[..., None, :], dim=-1)
    
    # Extract RGB and Sigma
    rgb = torch.sigmoid(raw[..., :3])
    sigma = raw[..., 3]
    
    # Add noise to sigma if specified
    if raw_noise_std > 0:
        noise = torch.randn_like(sigma) * raw_noise_std
        sigma = sigma + noise
        
    # Apply ReLU to sigma
    sigma = F.relu(sigma)
    
    # Compute alpha
    alpha = 1.0 - torch.exp(-sigma * dists)
    
    # Compute transmittance T
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # Equivalent to cumulative product of (1 - alpha)
    transmittance = torch.cumprod(torch.cat([torch.ones_like(alpha[..., :1]), 1.0 - alpha + 1e-10], dim=-1), dim=-1)
    T = transmittance[..., :-1]
    
    # Compute weights
    weights = alpha * T
    
    # Render RGB and Depth
    rgb_map = torch.sum(weights[..., None] * rgb, dim=-2)
    depth_map = torch.sum(weights * z_vals, dim=-1)
    acc_map = torch.sum(weights, dim=-1)
    
    # Handle white background
    if white_bkgd:
        rgb_map = rgb_map + (1.0 - acc_map[..., None])
        
    return rgb_map, depth_map, acc_map, weights, T

def render_rays_coarse_fine(rays_o, rays_d, near, far, n_coarse, n_fine, 
                            coarse_mlp, fine_mlp, perturb=True, raw_noise_std=0):
    """
    Implements the Coarse-to-Fine NeRF rendering pipeline.
    """
    # 1. Coarse Sampling
    # Uniform stratified sampling
    t_vals = torch.linspace(near, far, n_coarse, device=rays_o.device)
    t_vals = t_vals.expand(rays_o.shape[0], n_coarse)
    
    if perturb:
        mid = (t_vals[..., :-1] + t_vals[..., 1:]) / 2.0
        upper = torch.cat([mid, t_vals[..., -1:]], dim=-1)
        lower = torch.cat([t_vals[..., :1], mid], dim=-1)
        t_vals = lower + (upper - lower) * torch.rand_like(t_vals)
        
    # Query Coarse MLP
    pts_coarse = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * t_vals.unsqueeze(2)
    # Assume MLP takes (pts, dirs) and returns raw outputs
    raw_coarse = coarse_mlp(pts_coarse, rays_d.unsqueeze(1).expand_as(pts_coarse))
    
    # Render Coarse
    rgb_coarse, depth_coarse, acc_coarse, weights_coarse, _ = raw2outputs(
        raw_coarse, t_vals, rays_d, raw_noise_std
    )
    
    # 2. Hierarchical Sampling for Fine Pass
    # Compute midpoints for importance sampling
    t_mid = (t_vals[..., :-1] + t_vals[..., 1:]) / 2.0
    
    # Sample from weights_coarse using inverse transform sampling
    # Note: weights_coarse must be detached to break gradient flow
    weights_coarse_detached = weights_coarse.detach()
    t_vals_detached = t_vals.detach()
    
    # Simple inverse transform sampling from the discrete distribution
    # In practice, this often uses a helper like sample_pdf
    # Here we simulate the logic: draw uniform randoms and map to t_mid bins weighted by weights
    u = torch.rand_like(t_vals_detached)
    # Normalize weights to sum to 1 for each ray
    weights_norm = weights_coarse_detached / (weights_coarse_detached.sum(dim=-1, keepdim=True) + 1e-10)
    # Cumulative sum
    cumsum_weights = torch.cumsum(weights_norm, dim=-1)
    # Find indices where u falls
    indices = torch.searchsorted(cumsum_weights, u, side='right')
    # Clamp indices
    indices = torch.clamp(indices, 0, n_coarse - 2)
    # Interpolate within bins
    t_fine = t_mid.gather(1, indices) + torch.rand_like(t_vals) * (t_vals[..., 1:] - t_vals[..., :-1]).gather(1, indices)
    
    # 3. Concatenate and Sort
    t_fine_samples = torch.cat([t_vals, t_fine], dim=-1)
    t_fine_samples, _ = torch.sort(t_fine_samples, dim=-1)
    
    # 4. Query Fine MLP
    pts_fine = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * t_fine_samples.unsqueeze(2)
    raw_fine = fine_mlp(pts_fine, rays_d.unsqueeze(1).expand_as(pts_fine))
    
    # 5. Render Fine
    rgb_fine, depth_fine, acc_fine, weights_fine, _ = raw2outputs(
        raw_fine, t_fine_samples, rays_d, raw_noise_std
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
A variant that uses a single MLP with parameters $\Theta$ for both coarse and fine stages, enabled by Integrated Positional Encoding (IPE) which explicitly encodes scale into input features [S6]. Unlike standard NeRF which requires two distinct MLPs because PE features only model one scale, this approach allows one MLP to learn a multiscale representation [S6]. The optimization balances coarse and fine losses with a hyperparameter $\lambda$ (default 0.1) [S6].

### Segmented Sampling
A strategy that selectively increases the sampling rate only within the bounding box of an object area to ensure detailed sampling without significantly increasing overall computational cost [S7]. It addresses the pitfall where standard uniform sampling results in sparse points on small objects relative to the scene, causing parts of the object to fall between sampled points and leading to inaccurate PDFs in the fine stage [S7].

### DONeRF PDF Sampling
A method that translates depth oracle classification outputs into sample locations by treating the output vector as a piecewise-constant Probability Density Function (PDF) and sampling from its Cumulative Distribution Function (CDF) [S8]. Unlike NeRF, which builds a PDF from opacity outputs of a coarse shading network evaluated at multiple sample locations, DONeRF uses the output from a single oracle network evaluation [S8].

## Operating characteristics
*   **Time Complexity:** $O(N_C + N_F)$ MLP evaluations per ray, where $N_C$ and $N_F$ are the number of coarse and fine samples. Typically $N_C=64, N_F=128$ [S4, unofficial].
*   **Memory Complexity:** $O(N_C + N_F)$ per ray to store sample positions, raw outputs, and weights [S5, unofficial].
*   **Performance:** Improves rendering quality by focusing computational budget on opaque regions, reducing aliasing and missing details compared to uniform sampling [S1, S4, unofficial].
*   **Strengths:** Efficiently captures high-frequency details; reduces noise in rendered images [S1, S4, unofficial].
*   **Limitations:** Requires two network passes per ray; non-differentiable sampling step requires careful gradient management (detaching) [S4, S5, unofficial].
*   **When to use:** When high-quality detail is required and uniform sampling is too sparse or computationally prohibitive to increase density globally [S1, S4, unofficial].
*   **When not to use:** In extremely unbounded scenes where the far bound is very large, as uniform coarse sampling may miss all opaque regions if $N_C$ is small [S1, unofficial].

## Background
Coarse-to-Fine NeRF, also known as Hierarchical Rendering, was introduced in the original NeRF paper by Mildenhall et al. [S2][S5, unofficial]. It was designed to improve computational efficiency and rendering quality by using a coarse network to guide the sampling of a fine network [S2]. The method has become a standard component in many NeRF variants and acceleration techniques [S2].

## Sources
[S1] Fourier Feature Networks and Neural Volume Rendering, https://github.com/matajoh/fourier_feature_nets, fetched 2026-10-26 (unofficial)
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S4] yashtmy/nerf-3d-vision, https://github.com/yashtmy/nerf-3d-vision, fetched 2026-10-26 (unofficial)
[S5] NeRF: A Volume Rendering Perspective | Yue Yu - GitHub Pages, https://yconquesty.github.io/blog/ml/nerf/nerf_rendering.html, fetched 2026-10-29 (unofficial)
[S6] Mip-NeRF: A Multiscale Representation for Anti-Aliasing Neural Radiance Fields, https://doi.org/10.1109/iccv48922.2021.00580, fetched 2026-10-26
[S7] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
[S8] DONeRF: Towards Real‐Time Rendering of Compact Neural Radiance Fields using Depth Oracle Networks, https://doi.org/10.1111/cgf.14340, fetched 2026-10-26
