# Coarse-Fine Sampling

Also called: Coarse-Fine Rendering, Coarse/Fine Sampling Pipeline, Hierarchical NeRF Rendering, Hierarchical Rendering Pipeline, Importance Sampling Pipeline, Two-Pass NeRF, Two-Stage NeRF Sampling

## Overview
Coarse-Fine Sampling is a two-pass rendering technique for Neural Radiance Fields (NeRF) that improves image quality and sampling efficiency by first estimating the scene's density distribution with a coarse pass, then using that distribution to importance-sample a fine pass [S1, S2, S3, unofficial]. It sets itself apart from uniform sampling by concentrating computational effort on high-density regions identified in the first pass, and from single-stage methods by decoupling the rough density estimation from the detailed color/density evaluation [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Ray origins $\mathbf{o}$, ray directions $\mathbf{d}$, near/far clipping planes $t_{near}, t_{far}$, number of coarse samples $N_c$, number of fine samples $N_f$, perturbation flag `perturb`, and optional disparity sampling flag `use_disp` (or `lindisp`) [S1, S2, S3, unofficial].
- **Outputs:** Final RGB color $\mathbf{C}(\mathbf{r})$, accumulated opacity $\alpha(\mathbf{r})$, depth $D(\mathbf{r})$, and optionally the coarse RGB $\mathbf{C}_c(\mathbf{r})$ for auxiliary loss [S2, S3, unofficial].

**Data Structures:**
- **Samples:** Points $\mathbf{p}_i$ and depths $t_i$ along each ray.
- **Model Outputs:** Color $\mathbf{c}_i \in \mathbb{R}^3$ and density $\sigma_i \in \mathbb{R}$ for each sample point [S2, S3, unofficial].
- **Weights:** Transmittance $T_i$ and weight $w_i$ for volume rendering integration [S2, S3, unofficial].

**Parameters:**
- $N_c$: Number of coarse samples per ray. Default: 64 [S1, S2, S3, unofficial].
- $N_f$: Number of fine samples per ray. Default: 128 [S1, S2, S3, unofficial]. If $N_f=0$, the fine model is not used [S1, unofficial].
- `chunk`: Batch size for model inference to manage memory. Default: 1024 or $1024 \times 32$ depending on implementation [S2, S3, unofficial].
- `perturb`: Random noise addition to sample positions during training. Default: 1.0 (on) [S3, unofficial].
- `use_disp`/`lindisp`: Boolean flag for disparity-based depth sampling instead of uniform depth sampling. Default: False [S1, S3, unofficial].

**Steps:**

1. **Coarse Sampling:**
   Generate $N_c$ stratified samples along the ray interval $[t_{near}, t_{far}]$. If `lindisp` is true, sample linearly in disparity ($1/t$) [S2, S3, unofficial].
   $$ t_i = t_{near} + \frac{i + u}{N_c} (t_{far} - t_{near}) $$
   where $u \sim \mathcal{U}(0,1)$ if `perturb` is true, else $u = 0$ [S2, S3, unofficial].
   Compute sample positions:
   $$ \mathbf{p}_i = \mathbf{o} + t_i \mathbf{d} $$
   [S2, S3, unofficial].

2. **Coarse Evaluation:**
   Evaluate the coarse NeRF model $F_{\Theta_c}$ at each $\mathbf{p}_i$ to obtain color $\mathbf{c}_i^c$ and density $\sigma_i^c$ [S2, S3, unofficial]. Process in chunks of size `chunk` to avoid out-of-memory errors [S2, S3, unofficial].

3. **Coarse Volume Rendering:**
   Compute transmittance $T_i$ and weights $w_i$:
   $$ \hat{\sigma}_i = \max(\sigma_i^c, \epsilon) $$
   $$ \alpha_i = 1 - \exp(-\hat{\sigma}_i \delta_i) $$
   $$ T_i = \prod_{j < i} (1 - \alpha_j) $$
   $$ w_i = T_i \alpha_i $$
   where $\delta_i = t_{i+1} - t_i$ [S2, S3, unofficial].
   Render coarse color, depth, and accumulation:
   $$ \mathbf{C}_c(\mathbf{r}) = \sum_{i=1}^{N_c} w_i \mathbf{c}_i^c $$
   $$ D_c(\mathbf{r}) = \sum_{i=1}^{N_c} w_i t_i $$
   $$ A_c(\mathbf{r}) = \sum_{i=1}^{N_c} w_i $$
   [S2, S3, unofficial].

4. **Fine Sampling (Importance Sampling):**
   Use the coarse weights $w_i$ as a probability distribution to sample $N_f$ additional points [S2, S3, unofficial].
   Compute the cumulative distribution function (CDF) from $w_i$:
   $$ C_i = \sum_{j=1}^{i} w_j $$
   Sample $N_f$ uniform random values $u_k \sim \mathcal{U}(0,1)$ [S2, S3, unofficial].
   For each $u_k$, find the interval $[C_{i-1}, C_i]$ containing $u_k$ and interpolate the depth $t$ within that interval [S2, S3, unofficial].
   $$ t_k = t_{i-1} + \frac{u_k - C_{i-1}}{w_i} (t_i - t_{i-1}) $$
   [S2, S3, unofficial].
   If `perturb` is true, add noise to $t_k$ [S2, unofficial].

5. **Merge and Sort:**
   Combine coarse depths $t^c$ and fine depths $t^f$ into a single sorted list $t_{all}$ [S2, unofficial].
   $$ t_{all} = \text{sort}(\text{concat}(t^c, t^f)) $$
   [S2, unofficial].
   Compute corresponding points $\mathbf{p}_{all}$:
   $$ \mathbf{p}_{all, k} = \mathbf{o} + t_{all, k} \mathbf{d} $$
   [S2, unofficial].

6. **Fine Evaluation:**
   Evaluate the fine NeRF model $F_{\Theta_f}$ at each $\mathbf{p}_{all, k}$ to obtain $\mathbf{c}_k^f$ and $\sigma_k^f$ [S2, S3, unofficial]. Process in chunks [S2, S3, unofficial].

7. **Fine Volume Rendering:**
   Compute transmittance and weights for the merged samples:
   $$ \hat{\sigma}_k = \max(\sigma_k^f, \epsilon) $$
   $$ \alpha_k = 1 - \exp(-\hat{\sigma}_k \delta_k) $$
   $$ T_k = \prod_{j < k} (1 - \alpha_j) $$
   $$ w_k = T_k \alpha_k $$
   [S2, S3, unofficial].
   Render final color, depth, and accumulation:
   $$ \mathbf{C}(\mathbf{r}) = \sum_{k=1}^{N_c + N_f} w_k \mathbf{c}_k^f $$
   $$ D(\mathbf{r}) = \sum_{k=1}^{N_c + N_f} w_k t_{all, k} $$
   $$ A(\mathbf{r}) = \sum_{k=1}^{N_c + N_f} w_k $$
   [S2, S3, unofficial].

8. **Auxiliary Loss (Optional):**
   Attach the coarse RGB $\mathbf{C}_c(\mathbf{r})$ to the fine render output for use in auxiliary loss functions (e.g., MSE between coarse and fine RGB) [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import math

def stratified_sample(near, far, n_samples, perturb=True):
    """Generate stratified samples along a ray."""
    # Uniform grid
    t_vals = torch.linspace(near, far, n_samples)
    # Add jitter
    if perturb:
        u = torch.rand(n_samples)
        t_vals = t_vals + (u - 0.5) * (far - near) / n_samples
    else:
        t_vals = t_vals + (torch.arange(n_samples) + 0.5) / n_samples * (far - near) - (far - near) / 2
        # Correction for uniform stratified:
        # Standard implementation: t_vals = near + (far - near) * (i + u) / N
        # Re-implementing standard stratified:
        t_vals = near + (far - near) * (torch.arange(n_samples) + torch.rand(n_samples)) / n_samples
    return t_vals

def sample_hierarchical(z_vals_coarse, weights_coarse, n_samples_fine, perturb=True):
    """Importance sample fine points based on coarse weights."""
    # weights_coarse shape: (N_rays, N_c)
    # z_vals_coarse shape: (N_rays, N_c)
    
    # Get intervals
    num_rays = weights_coarse.shape[0]
    num_coarse = weights_coarse.shape[1]
    
    # Compute CDF
    # Add small epsilon to avoid division by zero
    eps = 1e-5
    weights_coarse = weights_coarse + eps
    cdf = torch.cumsum(weights_coarse, dim=-1)
    cdf = cdf / cdf[:, -1:]  # Normalize
    
    # Sample uniform values
    u = torch.rand(num_rays, n_samples_fine)
    
    # Inverse CDF sampling
    # Find indices where u falls in the CDF
    indices = torch.searchsorted(cdf, u, side='right')
    indices = torch.clamp(indices, 0, num_coarse - 1)
    
    # Get lower and upper bounds for interpolation
    lower_idx = indices - 1
    lower_idx = torch.clamp(lower_idx, 0, num_coarse - 1)
    
    z_low = torch.gather(z_vals_coarse, 1, lower_idx)
    z_high = torch.gather(z_vals_coarse, 1, indices)
    c_low = torch.gather(cdf, 1, lower_idx)
    c_high = torch.gather(cdf, 1, indices)
    
    # Interpolate
    t_vals = z_low + (z_high - z_low) * (u - c_low) / (c_high - c_low + eps)
    
    if perturb:
        # Add noise to break symmetry
        noise = torch.rand_like(t_vals) * (z_high - z_low) / n_samples_fine
        t_vals = t_vals + noise
        
    return t_vals

def volume_render(rgb, sigma, t_vals, ray_d, white_bg=False):
    """Standard volume rendering."""
    # rgb: (N_rays, N_samples, 3)
    # sigma: (N_rays, N_samples, 1)
    # t_vals: (N_rays, N_samples)
    
    # Compute deltas
    deltas = t_vals[:, 1:] - t_vals[:, :-1]
    # Handle last delta
    delta_inf = torch.ones_like(deltas[:, :1]) * 1e10
    deltas = torch.cat([deltas, delta_inf], dim=-1)
    
    # Compute alpha
    eps = 1e-5
    sigma = torch.clamp(sigma, min=eps)
    alpha = 1.0 - torch.exp(-sigma * deltas)
    
    # Compute transmittance
    T = torch.cumprod(1.0 - alpha + eps, dim=-1)
    T = torch.cat([torch.ones_like(T[:, :1]), T[:, :-1]], dim=-1)
    
    # Compute weights
    weights = alpha * T
    
    # Render RGB
    rgb_map = torch.sum(weights * rgb, dim=-2)
    
    # Render Depth
    depth_map = torch.sum(weights * t_vals, dim=-1)
    
    # Render Accumulation
    acc_map = torch.sum(weights, dim=-1)
    
    # Background blending
    if white_bg:
        rgb_map = rgb_map + (1.0 - acc_map[:, None]) * 1.0
        
    return {
        "rgb_map": rgb_map,
        "depth_map": depth_map,
        "acc_map": acc_map,
        "weights": weights
    }

def render_rays(model, ray_o, ray_d, near, far, N_c, N_f, perturb=True, white_bg=False, chunk=1024*32):
    """Two-stage rendering pipeline."""
    # Coarse Pass
    z_vals_coarse = stratified_sample(near, far, N_c, perturb)
    pts_coarse = ray_o.unsqueeze(1) + ray_d.unsqueeze(1) * z_vals_coarse.unsqueeze(-1)
    
    # Evaluate coarse model in chunks
    rgb_c, sigma_c = model.forward_coarse(pts_coarse, chunk=chunk)
    
    # Render coarse
    render_c = volume_render(rgb_c, sigma_c, z_vals_coarse, ray_d, white_bg)
    
    # Fine Pass
    if N_f > 0:
        z_vals_fine = sample_hierarchical(z_vals_coarse, render_c["weights"].detach(), N_f, perturb)
        
        # Merge and sort
        z_vals_all = torch.cat([z_vals_coarse, z_vals_fine], dim=-1)
        z_vals_all, _ = torch.sort(z_vals_all, dim=-1)
        
        pts_fine = ray_o.unsqueeze(1) + ray_d.unsqueeze(1) * z_vals_all.unsqueeze(-1)
        
        # Evaluate fine model in chunks
        rgb_f, sigma_f = model.forward_fine(pts_fine, chunk=chunk)
        
        # Render fine
        render_f = volume_render(rgb_f, sigma_f, z_vals_all, ray_d, white_bg)
        
        # Attach coarse RGB for auxiliary loss
        render_f["rgb_coarse"] = render_c["rgb_map"]
        
        return render_f
    else:
        return render_c
```

**Edge Cases and Pitfalls:**
- **Zero Density:** If all coarse densities are zero, importance sampling fails. Add a small epsilon to weights before computing CDF [S2, S3, unofficial].
- **Memory Limits:** Large batches or high $N_c, N_f$ can cause OOM. Use chunked evaluation for model forward passes [S2, S3, unofficial].
- **Disparity Sampling:** For forward-facing scenes, `lindisp` improves depth resolution near the camera. For general scenes, uniform sampling is preferred [S1, S3, unofficial].
- **Perturbation:** Disabling perturbation during inference improves stability but may reduce training quality if not used during training [S3, unofficial].

## Subtopics

### Combined Sampling (Occupancy Grid + Proposal Network)
A method that uses an occupancy grid to quickly identify empty space and a proposal network to sample high-density regions, often used in conjunction with or as an alternative to coarse-fine sampling for further acceleration. [Link](1626-combined-sampling-occupancy-grid-proposal-network.md)

### VF-NeRF Rendering Pipeline
A pipeline that integrates view-dependent features and potentially different sampling strategies, often optimizing for specific view-dependent effects or performance characteristics distinct from standard coarse-fine sampling. [Link](3302-vf-nerf-rendering-pipeline.md)

## Operating characteristics
- **Time Complexity:** $O((N_c + N_f) \cdot N_{rays})$ for evaluation, where $N_{rays}$ is the number of rays processed in a batch. The coarse pass is $O(N_c \cdot N_{rays})$ and the fine pass is $O(N_f \cdot N_{rays})$ [S2, S3, unofficial].
- **Memory Complexity:** $O((N_c + N_f) \cdot N_{rays} \cdot C)$, where $C$ is the channel size for RGB and sigma. Chunked evaluation reduces peak memory usage [S2, S3, unofficial].
- **Measured Performance:** Typical values are $N_c=64, N_f=128$. This configuration provides a good balance between quality and speed, significantly improving over uniform sampling with $N=128$ by focusing samples on relevant geometry [S1, S2, S3, unofficial].
- **Strengths:** Improves rendering quality by concentrating samples in high-density regions; reduces noise compared to uniform sampling; allows for efficient use of computational resources [S2, S3, unofficial].
- **Limitations:** Requires two model evaluations (coarse and fine); importance sampling adds overhead; may not handle transparent surfaces as well as other methods without additional modifications [S2, S3, unofficial].
- **When to Use:** When high-quality rendering is needed and computational resources allow for two passes; when the scene has complex geometry or varying density [S2, S3, unofficial].
- **When Not to Use:** When real-time performance is critical and $N_f$ must be very small; when the scene is uniformly dense and importance sampling provides little benefit [S2, S3, unofficial].

## Background
Coarse-Fine Sampling was introduced in the original NeRF paper by Mildenhall et al. (2020) as a key technique for efficient volume rendering [S1, S2, S3, unofficial]. It has since become a standard component in many NeRF implementations and variants [S2, S3, unofficial]. The method builds on the idea of importance sampling in computer graphics, adapting it to the context of neural volumetric rendering [S2, S3, unofficial].

## Sources
[S1] kwea123/nerf_pl, https://github.com/kwea123/nerf_pl, fetched 2026-10-26 (unofficial)
[S2] emmanueladutwum123/nerf-from-scratch, https://github.com/emmanueladutwum123/nerf-from-scratch, fetched 2026-10-26 (unofficial)
[S3] damn8daniel/nerf-from-scratch, https://github.com/damn8daniel/nerf-from-scratch, fetched 2026-10-26 (unofficial)
