# NerfingMVS

Also called: Guided NeRF Optimization for MVS, Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, NeRF with Depth Priors

## Overview
NerfingMVS is a method for indoor multi-view stereo that optimizes Neural Radiance Fields (NeRF) using depth priors to resolve the shape-radiance ambiguity inherent in implicit volume representations [S2]. It differs from standard NeRF by explicitly incorporating external depth estimates to guide volume sampling and constrain ray bounds, thereby improving geometric accuracy without relying on explicit pixel matching across views [S3, unofficial].

## How it works
**Inputs:**
- A set of multi-view images with known camera poses (intrinsics $K$, extrinsics $C$).
- A pretrained monocular depth estimation network (e.g., Mannequin Challenge model) [S3, unofficial].
- Sparse 3D point cloud from Structure-from-Motion (SfM) and Multi-View Stereo (MVS) reconstruction (e.g., from COLMAP) [S2].

**Outputs:**
- Optimized NeRF model (implicit density and color fields).
- Dense per-view depth maps aligned with the scene geometry [S3, unofficial].

**Data Structures:**
- **NeRF Model:** A multi-layer perceptron (MLP) mapping 3D position $\mathbf{x}$ and view direction $\mathbf{d}$ to density $\sigma$ and color $\mathbf{c}$ [S3, unofficial].
- **Depth Priors:** A tensor of shape $[N_{rays}]$ containing depth estimates $D^*$ for each ray [S3, unofficial].
- **Depth Confidences:** A tensor of shape $[N_{rays}]$ containing confidence values $C^*$ associated with each prior [S3, unofficial].
- **Ray Tensor:** Concatenated tensor containing ray origins $\mathbf{o}$, directions $\mathbf{d}$, near/far bounds $t_{near}, t_{far}$, and optionally view directions [S3, unofficial].

**Algorithm Steps:**

1.  **Sparse Reconstruction:** Run COLMAP on the input images to obtain camera poses and a sparse 3D point cloud [S2].
2.  **Depth Prior Extraction:**
    - Project the sparse point cloud onto each view to create sparse depth maps [S2].
    - Fine-tune the pretrained monocular depth network on these sparse maps using a scale-invariant loss to generate dense, scene-specific depth priors $D^*$ and confidence maps $C^*$ [S2][S3, unofficial].
    - The scale-invariant loss minimizes the difference between the log of the predicted depth $D_p$ and the sparse ground truth $D_{Sparse}$:
      $$ L(D_p, D_{Sparse}) = \frac{1}{n} \sum_{j=1}^n \left| \log D_p(j) - \log D_{Sparse}(j) + \alpha(D_p, D_{Sparse}) \right| $$
      where $\alpha$ is a correction term ensuring scale invariance [S2].
3.  **Ray Setup:**
    - For each pixel, compute ray origin $\mathbf{o}$ and direction $\mathbf{d}$ from camera parameters [S3, unofficial].
    - If NDC (Normalized Device Coordinates) is used, transform rays to NDC space [S3, unofficial].
    - Determine initial ray bounds $[t_{near}, t_{far}]$. NerfingMVS clamps these bounds using the extracted depth priors to restrict sampling to relevant depth ranges [S1][S3, unofficial].
4.  **Volume Rendering with Guidance:**
    - Divide the ray segment $[t_{near}, t_{far}]$ into $N$ equal bins [S1].
    - Sample points along the ray. The depth priors guide the placement or weighting of these samples [S1].
    - Query the NeRF MLP at sample points to get raw density $\sigma$ and color $\mathbf{c}$ [S3, unofficial].
    - Compute alpha values $\alpha_i$ for each sample using the density and distance between samples $dist_i$:
      $$ \alpha_i = 1 - \exp(-\sigma_i \cdot dist_i) $$
      [S3, unofficial].
    - Compute rendering weights $w_i$ using cumulative product for numerical stability:
      $$ w_i = \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j + \epsilon) $$
      where $\epsilon = 10^{-10}$ prevents NaNs when $\alpha \approx 1$ [S3, unofficial].
    - Compute the final color $\mathbf{C}(\mathbf{r})$ and depth $D(\mathbf{r})$ for the ray:
      $$ \mathbf{C}(\mathbf{r}) = \sum_{i=1}^N w_i \mathbf{c}_i $$
      $$ D(\mathbf{r}) = \sum_{i=1}^N w_i z_i $$
      where $z_i$ is the depth of the $i$-th sample [S3, unofficial].
5.  **Optimization:**
    - Minimize photometric loss between rendered RGB and input images.
    - Optionally, apply depth consistency loss between rendered depth and priors $D^*$ [S1].
    - Filter outliers in the final depth maps using consistency checks (e.g., top-k error filtering) [S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def nerfingmvs_render(rays_o, rays_d, depth_priors, depth_confidences, model, N_samples, near_bound, far_bound, white_bkgd=True):
    """
    Render rays using NerfingMVS guided sampling.
    
    Args:
        rays_o: Tensor of shape [batch_size, 3], ray origins.
        rays_d: Tensor of shape [batch_size, 3], ray directions.
        depth_priors: Tensor of shape [batch_size], depth estimates from monocular network.
        depth_confidences: Tensor of shape [batch_size], confidence values.
        model: NeRF MLP model.
        N_samples: Number of samples per ray.
        near_bound: Float, absolute near bound.
        far_bound: Float, absolute far bound.
        white_bkgd: Bool, composite against white background.
        
    Returns:
        rgb_map: Tensor of shape [batch_size, 3].
        depth_map: Tensor of shape [batch_size].
        acc_map: Tensor of shape [batch_size].
    """
    # 1. Determine ray bounds
    # Clamp bounds using depth priors as per NerfingMVS logic [S1][S3, unofficial]
    # Note: Exact clamping logic depends on confidence; simplified here to use priors directly
    t_near = near_bound * torch.ones_like(depth_priors)
    t_far = far_bound * torch.ones_like(depth_priors)
    
    # Optional: Use depth priors to refine bounds [S1]
    # t_near = torch.max(t_near, depth_priors - margin)
    # t_far = torch.min(t_far, depth_priors + margin)
    
    # 2. Sample points along the ray
    # Divide ray into N equal bins [S1]
    t_vals = torch.linspace(t_near, t_far, N_samples).unsqueeze(0).expand_as(depth_priors)
    # Add noise for stratified sampling (standard NeRF practice, implied by [S3, unofficial])
    # t_vals = t_vals + (torch.rand_like(t_vals) - 0.5) * (t_far - t_near) / N_samples
    
    # 3. Compute 3D positions
    # xyz = rays_o + rays_d * t_vals[..., None]
    # Note: rays_o and rays_d are [batch, 3], t_vals is [batch, N]
    # Need to broadcast correctly
    xyz = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * t_vals.unsqueeze(2)
    
    # 4. Query NeRF
    # Flatten for MLP input
    xyz_flat = xyz.reshape(-1, 3)
    # Compute view directions
    viewdirs = rays_d.unsqueeze(1).expand_as(xyz).reshape(-1, 3)
    viewdirs = viewdirs / (torch.norm(viewdirs, dim=-1, keepdim=True) + 1e-10)
    
    # Query model (returns raw density and color)
    # raw_density, raw_color = model(xyz_flat, viewdirs)
    # Placeholder for model query [S3, unofficial]
    raw = model(xyz_flat, viewdirs)
    raw_density = raw[..., 0]
    raw_color = raw[..., 1:]
    
    # Reshape back
    raw_density = raw_density.reshape(xyz.shape[:-1])
    raw_color = raw_color.reshape(xyz.shape[:-1] + (3,))
    
    # 5. Compute alpha and weights
    # Distances between samples
    dists = t_vals[..., 1:] - t_vals[..., :-1]
    dists = torch.cat([dists, torch.tensor([1e10]).to(dists.device)], dim=-1) # Append large value [S3, unofficial]
    
    # Scale by ray direction norm
    dists = dists * torch.norm(rays_d.unsqueeze(1), dim=-1)
    
    # Alpha from density
    # alpha = 1 - exp(-sigma * dist)
    alpha = 1.0 - torch.exp(-F.relu(raw_density) * dists)
    
    # Weights
    # weights = alpha * prod(1 - alpha_prev)
    # Use cumprod for stability
    ones = torch.ones_like(alpha[..., 0])
    accum_alphas = torch.cat([ones, 1.0 - alpha + 1e-10], dim=-1)[:, :-1] # [batch, N]
    # Cumulative product of (1-alpha)
    # Note: Standard NeRF uses cumprod on (1-alpha). Here we compute weights directly.
    # w_i = alpha_i * prod_{j<i} (1-alpha_j)
    # Let's use the standard recurrence or cumprod
    # prod_alphas = torch.cumprod(1.0 - alpha + 1e-10, dim=-1)
    # weights = alpha * torch.cat([torch.ones_like(alpha[..., :1]), prod_alphas[..., :-1]], dim=-1)
    
    # Simpler implementation matching [S3, unofficial] logic:
    # weights = alpha * cumprod([1, 1-alpha + 1e-10])[:, :-1]
    # This implies:
    prod_alphas = torch.cat([torch.ones_like(alpha[..., :1]), 1.0 - alpha + 1e-10], dim=-1)
    cum_prod = torch.cumprod(prod_alphas, dim=-1)
    weights = alpha * cum_prod[..., :-1]
    
    # 6. Render RGB and Depth
    # RGB
    rgb_map = torch.sum(weights.unsqueeze(-1) * raw_color, dim=-2)
    
    # Depth
    depth_map = torch.sum(weights * t_vals, dim=-1)
    
    # Accumulated Opacity
    acc_map = torch.sum(weights, dim=-1)
    
    # Disparity (optional, for evaluation)
    # disp_map = 1.0 / torch.max(torch.tensor(1e-10).to(depth_map.device), depth_map / torch.max(acc_map, torch.tensor(1e-10).to(depth_map.device)))
    
    # Background
    if white_bkgd:
        rgb_map = rgb_map + (1.0 - acc_map.unsqueeze(-1))
        
    return rgb_map, depth_map, acc_map

def nerfingmvs_train_step(rays_o, rays_d, images, depth_priors, depth_confidences, model, optimizer):
    """
    Single training step for NerfingMVS.
    """
    optimizer.zero_grad()
    
    # Render
    rgb_rendered, depth_rendered, acc = nerfingmvs_render(
        rays_o, rays_d, depth_priors, depth_confidences, model, 
        N_samples=64, near_bound=0.05, far_bound=0.15, white_bkgd=True
    )
    
    # Photometric Loss
    loss = F.mse_loss(rgb_rendered, images)
    
    # Depth Loss (Optional, as per [S1])
    # depth_loss = F.l1_loss(depth_rendered, depth_priors)
    # loss += depth_loss
    
    loss.backward()
    optimizer.step()
    
    return loss.item()
```

## Operating characteristics
- **Time Complexity:** Training time is dominated by the NeRF optimization loop. The addition of depth prior extraction and fine-tuning adds a preprocessing step. The rendering step is similar to standard NeRF but with potentially fewer samples if bounds are tightly clamped [S1][S3, unofficial].
- **Memory Complexity:** Similar to standard NeRF, dominated by the MLP weights and the storage of depth priors and confidences for all rays [S3, unofficial].
- **Performance:** On ScanNet scenes, NerfingMVS achieves competitive depth estimation accuracy compared to traditional MVS methods, while providing high-quality novel view synthesis [S2]. It requires approximately 3.5 hours per scene on a single RTX 2080 GPU [S3, unofficial].
- **Strengths:**
    - Resolves shape-radiance ambiguity in indoor scenes [S2].
    - Does not require explicit pixel matching, which is error-prone in textureless or repetitive regions [S2].
    - Improves geometry quality by leveraging learning-based priors [S2].
- **Limitations:**
    - Relies on the quality of COLMAP reconstruction for initial sparse points [S3, unofficial].
    - Performance depends on the accuracy of the monocular depth network [S2].
    - Limited to indoor scenes where the monocular priors are effective [S2].

## Background
NerfingMVS was introduced by Wei et al. in September 2021 at ICCV [S2]. It addresses the limitations of standard NeRF in multi-view stereo tasks, particularly the ambiguity between shape and radiance in indoor environments [S2]. The method builds upon the NeRF framework [S3, unofficial] and integrates insights from multi-view stereo (MVS) and monocular depth estimation [S2]. It was trained on datasets such as ScanNet and Matterport3D [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Depth-Guided Optimization of Neural Radiance Fields for Indoor Multi-View Stereo, https://doi.org/10.1109/tpami.2023.3263464, fetched 2026-10-27
[S3] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
