# NerfAcc Rendering Pipeline

Also called: Efficient Sampling for Volumetric Rendering, NeRF Volume Rendering (CPU/PyTorch), NerfAcc, Neural Radiance Fields (NeRF) with Coarse/Fine Sampling and NDC, Neural Volume Rendering (Coarse/Fine Sampling), Volume Rendering with Neural Fields

## Overview
The NerfAcc Rendering Pipeline is a modular, high-performance volumetric rendering method implemented as a PyTorch library [S1]. It decouples the sampling strategy from the radiance field representation, supporting both Occupancy Grids and Proposal Networks to accelerate training and inference [S1]. Unlike standard uniform sampling, it uses interval-based ray representation to support conical frustums and integrates coarse/fine sampling with importance sampling to concentrate samples on surfaces while skipping empty space [S1][S3, unofficial]. It builds on [NeRF Volumetric Rendering (Coarse/Fine Sampling)](979-nerf-volumetric-rendering-coarse-fine-sampling.md) by providing a unified API for efficient transmittance estimation and differentiable rendering.

## How it works
**Inputs:**
- Ray origins $r_o \in \mathbb{R}^{N_{rays} \times 3}$ and normalized directions $r_d \in \mathbb{R}^{N_{rays} \times 3}$.
- Near and far clipping planes $near, far$.
- Neural radiance field model (coarse and optionally fine) providing density $\sigma$ and color $c$.
- Transmittance estimator (Occupancy Grid or Proposal Network).

**Outputs:**
- Rendered RGB color $C$, opacity $\alpha$, depth $z$, and auxiliary data (variance, weights).

**Data Structures:**
- **Packed Samples:** Instead of individual points, samples are stored as intervals $(t_0, t_1, r_{id})$ where $t_0, t_1$ are start/end distances along the $r_{id}$-th ray. This allows conical sampling support [S1].
- **Ray Array:** Structure of shape $(N_{rays}, 8)$. Columns 0-2: origin $(x,y,z)$; Columns 3-5: direction $(x,y,z)$; Column 6: near depth; Column 7: far depth [S3, unofficial].

**Parameters:**
- $N_{samples}$: Number of coarse samples per ray (default 64) [S3, unofficial].
- $N_{importance}$: Number of fine samples (default 0) [S3, unofficial].
- $near, far$: Clipping bounds (typical: 0.5, 3.0) [S3, unofficial].
- $perturb$: Jitter factor for coarse sampling (default 0) [S3, unofficial].
- $noise\_std$: Noise added to $\sigma$ (default 1) [S3, unofficial].
- $chunk$: Batch size for network inference (default 32768) [S3, unofficial].
- $white\_back$: Boolean, adds background weight to RGB if true [S3, unofficial].
- $use\_disp$: Boolean, uses inverse depth interpolation if true [S3, unofficial].

**Steps:**

1. **Ray Preparation:**
   Compute ray directions in camera coordinates using pixel grid indices $(i, j)$, image dimensions $(H, W)$, and focal length $f$:
   $$ d_{cam} = \left[ \frac{i - W/2}{f}, -\frac{j - H/2}{f}, -1 \right] $$
   Transform to world coordinates: $r_d = \text{normalize}(d_{cam} \cdot R_{c2w})$, $r_o = T_{c2w}$ [S3, unofficial].
   If the scene is unbounded, transform rays to Normalized Device Coordinates (NDC). Shift origins to the near plane: $t = -(near + r_o \cdot z) / r_d \cdot z$, then $r_o = r_o + t \cdot r_d$. Project to $[-1, 1]$ space [S3, unofficial].

2. **Coarse Sampling:**
   Generate initial sample depths $z_{vals}$ in $[0, 1]$ using logspace:
   $$ z_{vals} = \text{logspace}(0, 1, N_{samples}, \text{base}=2.0) - 1 $$
   Map to $[near, far]$: $z_{vals} = near + (far - near) \cdot z_{vals}$ [S1][S3, unofficial].
   If $use\_disp$ is true, use inverse interpolation: $z_{vals} = 1 / (1/near \cdot (1-z_{steps}) + 1/far \cdot z_{steps})$ [S3, unofficial].
   If $perturb > 0$, add jitter:
   $$ z_{vals\_mid} = 0.5 \cdot (z_{vals}[:, :-1] + z_{vals}[:, 1:]) $$
   $$ z_{vals} = z_{vals\_mid} + (z_{vals\_mid} - z_{vals}[:, :-1]) \cdot \text{random\_jitter} $$
   [S3, unofficial].

3. **Efficient Sampling (NerfAcc Core):**
   Define density function $density\_fn(t_0, t_1, r_{id})$ querying the neural network [S1].
   Initialize transmittance estimator: $estimator = \text{nerfacc.TransmittanceEstimator()}$ [S1].
   Perform efficient sampling to get packed intervals:
   $$ t_0, t_1, r_{id} = \text{nerfacc.sampling}(r_o, r_d, estimator, density\_fn) $$
   This step uses the estimator to skip empty space (Occupancy Grid) or concentrate samples (Proposal Network) [S1].

4. **Coarse Rendering:**
   Query the neural network for $\sigma$ and $c$ at sample points. To manage memory, chunk the inference if batch size $> chunk$ [S3, unofficial].
   Compute deltas: $\delta_i = t_{i+1} - t_i$. Append infinity for the last segment: $\delta_{last} = \infty$ [S3, unofficial].
   Convert to real-world distance: $\delta_i = \delta_i \cdot \|r_d\|$ [S3, unofficial].
   Add noise to density: $\sigma' = \text{relu}(\sigma + \mathcal{N}(0, noise\_std))$ [S3, unofficial].
   Compute opacity:
   $$ \alpha_i = 1 - \exp(-\delta_i \sigma'_i) $$
   [S2, S3, unofficial].
   Compute weights:
   $$ \alpha_{shifted} = [1, 1-\alpha_1+\epsilon, \dots, 1-\alpha_{N-1}+\epsilon] $$
   $$ w_i = \alpha_i \cdot \prod_{j=0}^{i-1} (1-\alpha_j) $$
   [S2, S3, unofficial].
   Compute color and depth:
   $$ C_{coarse} = \sum w_i c_i, \quad z_{coarse} = \sum w_i \frac{t_{i+1}+t_i}{2} $$
   [S2, S3, unofficial].
   If $white\_back$ is true: $C_{coarse} = C_{coarse} + (1 - \sum w_i) \cdot 1$ [S3, unofficial].

5. **Fine Sampling (if $N_{importance} > 0$):**
   Compute midpoints of coarse intervals: $z_{mid} = 0.5 \cdot (z_{vals}[:, :-1] + z_{vals}[:, 1:])$ [S3, unofficial].
   Perform stratified sampling (PDF sampling) of $N_{importance}$ points using coarse weights $w_{coarse}$ as probability distribution. Detach gradients to prevent backprop to coarse weights [S3, unofficial].
   Handle zero-weight bins by adding $\epsilon$ to weights before PDF computation [S3, unofficial].
   Combine coarse and fine $z_{vals}$, sort them, and compute 3D positions [S3, unofficial].

6. **Fine Rendering:**
   Query fine neural model with fine samples to get $c_{fine}, \sigma_{fine}$ [S3, unofficial].
   Compute new deltas, alphas, and weights using the same formulas as coarse rendering [S2, S3, unofficial].
   Compute final color, depth, and opacity [S3, unofficial].

7. **Update Estimator:**
   Update the transmittance estimator every $n$ steps using the sampled intervals and auxiliary data:
   $$ estimator.update\_every\_n\_steps(t_0, t_1, r_{id}, aux) $$
   [S1].

8. **Loss and Backprop:**
   Compute loss (e.g., MSE) between rendered color and ground truth, and backpropagate [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import nerfacc

def nerfacc_rendering_pipeline(r_o, r_d, near, far, nerf_model, estimator, 
                               N_samples=64, N_importance=0, perturb=0.0, 
                               noise_std=1.0, white_back=False, chunk=32768):
    """
    r_o: (N_rays, 3) ray origins
    r_d: (N_rays, 3) ray directions (normalized)
    nerf_model: object with .forward(r_o, r_d, t0, t1) returning (rgb, sigma)
    estimator: nerfacc.TransmittanceEstimator instance
    """
    
    # 1. Define functions for NerfAcc
    def density_fn(t0, t1, r_id):
        # Query sigma from nerf model
        # Note: nerf_model.forward should handle chunking internally or be wrapped
        rgb, sigma = nerf_model.forward(r_o[r_id], r_d[r_id], t0, t1)
        return sigma

    def rgb_density_fn(t0, t1, r_id):
        rgb, sigma = nerf_model.forward(r_o[r_id], r_d[r_id], t0, t1)
        return rgb, sigma

    # 2. Efficient Sampling using NerfAcc
    # Returns packed intervals: t0, t1, r_id
    t0, t1, r_id = nerfacc.sampling(r_o, r_d, estimator, density_fn=density_fn)

    # 3. Differentiable Volumetric Rendering
    # Returns color, opacity, depth, aux (weights, etc.)
    color, opacity, depth, aux = nerfacc.rendering(
        t0, t1, r_id, rgb_density_fn=rgb_density_fn
    )

    # 4. Update Estimator (typically called every N steps during training)
    # estimator.update_every_n_steps(t0, t1, r_id, aux)

    return color, opacity, depth, aux

# If implementing coarse/fine manually without NerfAcc's internal fine sampling:
def manual_coarse_fine(r_o, r_d, near, far, nerf_model, N_samples=64, N_importance=0, perturb=0.0):
    # Coarse Sampling
    z_vals = torch.logspace(0, 1, N_samples, base=2.0, device=r_o.device).unsqueeze(0).unsqueeze(0) - 1
    z_vals = near + (far - near) * z_vals
    
    # Perturbation
    if perturb > 0:
        z_vals_mid = 0.5 * (z_vals[:, :, :-1] + z_vals[:, :, 1:])
        lower = torch.cat([z_vals[:, :, :1], z_vals_mid], -1)
        upper = torch.cat([z_vals_mid, z_vals[:, :, -1:]], -1)
        z_vals = lower + (upper - lower) * torch.rand(z_vals.shape)

    # Query Coarse Model
    pts = r_o.unsqueeze(1) + r_d.unsqueeze(1) * z_vals.unsqueeze(-1)
    # Chunked inference for memory management
    rgbs_coarse, sigmas_coarse = nerf_model.forward(pts, r_d) # Shape: (N_rays, N_samples, 3/1)
    
    # Compute Deltas
    deltas = z_vals[:, :, 1:] - z_vals[:, :, :-1]
    delta_inf = 1e10 * torch.ones_like(deltas[:, :, :1])
    deltas = torch.cat([deltas, delta_inf], -1)
    deltas = deltas * torch.norm(r_d.unsqueeze(1), dim=-1)
    
    # Compute Alphas
    noise = torch.randn(sigmas_coarse.shape, device=sigmas_coarse.device) * noise_std
    alphas = 1 - torch.exp(-deltas * torch.relu(sigmas_coarse + noise))
    
    # Compute Weights
    alphas_shifted = torch.cat([torch.ones_like(alphas[:, :, :1]), 1 - alphas + 1e-10], -1)
    weights = alphas * torch.cumprod(alphas_shifted, -1)[:, :, :-1]
    
    # Coarse Color/Depth
    color_coarse = torch.sum(weights.unsqueeze(-1) * rgbs_coarse, dim=-2)
    z_mid = z_vals[:, :, :-1] + 0.5 * deltas
    depth_coarse = torch.sum(weights * z_mid, dim=-1)
    
    if N_importance > 0:
        # Fine Sampling
        z_vals_mid = z_vals[:, :, :-1] + 0.5 * deltas
        # Sample PDF
        weights_flat = weights.reshape(-1, N_samples)
        z_vals_fine = sample_pdf(z_vals_mid.reshape(-1, N_samples-1), 
                                 weights_flat[:, 1:-1].reshape(-1, N_samples-2) + 1e-5, 
                                 N_importance, det=(perturb==0))
        z_vals_fine = z_vals_fine.detach() # Stop gradient to coarse
        
        # Combine and Sort
        z_vals_combined = torch.cat([z_vals, z_vals_fine.unsqueeze(-1)], -1)
        z_vals_combined, _ = torch.sort(z_vals_combined, dim=-1)
        
        # Query Fine Model
        pts_fine = r_o.unsqueeze(1) + r_d.unsqueeze(1) * z_vals_combined.unsqueeze(-1)
        rgbs_fine, sigmas_fine = nerf_model.forward(pts_fine, r_d)
        
        # Re-compute Deltas/Alphas/Weights for fine samples
        deltas_fine = z_vals_combined[:, :, 1:] - z_vals_combined[:, :, :-1]
        delta_inf_fine = 1e10 * torch.ones_like(deltas_fine[:, :, :1])
        deltas_fine = torch.cat([deltas_fine, delta_inf_fine], -1)
        deltas_fine = deltas_fine * torch.norm(r_d.unsqueeze(1), dim=-1)
        
        alphas_fine = 1 - torch.exp(-deltas_fine * torch.relu(sigmas_fine))
        alphas_shifted_fine = torch.cat([torch.ones_like(alphas_fine[:, :, :1]), 1 - alphas_fine + 1e-10], -1)
        weights_fine = alphas_fine * torch.cumprod(alphas_shifted_fine, -1)[:, :, :-1]
        
        color_fine = torch.sum(weights_fine.unsqueeze(-1) * rgbs_fine, dim=-2)
        depth_fine = torch.sum(weights_fine * (z_vals_combined[:, :, :-1] + 0.5 * deltas_fine), dim=-1)
        
        return color_fine, depth_fine
    else:
        return color_coarse, depth_coarse

def sample_pdf(z_vals_mid, weights, N_importance, det=False):
    # Stratified sampling from PDF
    eps = 1e-5
    weights = weights + eps
    pdf = weights / torch.sum(weights, dim=-1, keepdim=True)
    cdf = torch.cumsum(pdf, dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[:, :1]), cdf], dim=-1)
    
    if det:
        u = torch.linspace(0, 1, steps=N_importance).to(weights.device)
        u = u.unsqueeze(0).expand(cdf.shape)
    else:
        u = torch.rand(cdf.shape).to(weights.device)
        
    u = u.contiguous()
    indices = torch.searchsorted(cdf, u, right=True)
    # Clamp indices
    indices = torch.clamp(indices, 0, cdf.shape[-1] - 2)
    
    # Linear interpolation
    low = indices - 1
    high = indices
    v_low = z_vals_mid[low]
    v_high = z_vals_mid[high]
    c_low = cdf[low]
    c_high = cdf[high]
    
    denom = c_high - c_low
    denom[denom < eps] = 1
    t = (u - c_low) / denom
    z_samples = v_low + t * (v_high - v_low)
    
    return z_samples
```

## Subtopics

### NeRF Upsampling (Fine Sampling)
A variant that explicitly performs importance sampling after coarse rendering to refine details. It computes weights from coarse samples, samples new depths from the resulting PDF, and queries the network again. It detaches gradients from the coarse weights to stabilize training [S2, S3, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N_{rays} \cdot N_{samples})$ for coarse, plus $O(N_{rays} \cdot N_{importance})$ for fine. The estimator update is $O(N_{samples})$.
- **Memory Complexity:** $O(N_{rays} \cdot N_{samples})$ for storing intervals and activations. Chunking reduces peak memory for network inference [S3, unofficial].
- **Performance:** Speedups of 1.5× to 20× over standard NeRF rendering depending on the scene and estimator used [S1].
- **Strengths:** Decoupled sampling allows easy integration of various acceleration structures (Occupancy Grid, Proposal Network). Interval-based representation supports anti-aliasing (Mip-NeRF) [S1].
- **Limitations:** Requires careful tuning of $near/far$ bounds and estimator parameters. NDC transformation is required for unbounded scenes [S3, unofficial].
- **When to use:** When training speed is critical or scenes have large empty spaces. Not suitable if strict analytical integration is required without sampling [S1].

## Background
NerfAcc was introduced in 2023 by Liu et al. as a toolbox for accelerating Neural Radiance Fields [S1]. It builds on the observation that Occupancy Grids (from Instant-NGP) and Proposal Networks (from Mip-NeRF 360) are complementary: one skips empty space, the other concentrates on surfaces [S1]. The library provides a unified PyTorch interface for these techniques, enabling efficient sampling and rendering pipelines [S1].

## Sources
[S1] NerfAcc: Efficient Sampling Accelerates NeRFs, https://doi.org/10.1109/iccv51070.2023.01699, fetched 2026-10-26
[S2] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S3] GitHub - nitthilan/neural_sparse_voxel_field: Neural scene ..., https://github.com/nitthilan/neural_sparse_voxel_field, fetched 2026-10-28 (unofficial)
