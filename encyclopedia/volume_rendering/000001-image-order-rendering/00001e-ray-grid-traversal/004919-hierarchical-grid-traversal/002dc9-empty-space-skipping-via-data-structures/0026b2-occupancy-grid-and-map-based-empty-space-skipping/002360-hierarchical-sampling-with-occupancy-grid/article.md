# Hierarchical Sampling with Occupancy Grid

Also called: FLASH Renderer, NeRF Hierarchical Sampling

## Overview
Hierarchical Sampling with Occupancy Grid is a two-stage volume rendering method that combines empty-space skipping via an occupancy grid with importance-based hierarchical sampling to accelerate Neural Radiance Field (NeRF) training and inference [S1, unofficial]. It differs from its siblings by explicitly decoupling the coarse sampling phase (guided by spatial occupancy) from the fine sampling phase (guided by rendered weights), allowing for efficient refinement of high-density regions without re-evaluating empty space [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origins $\mathbf{o}$, ray directions $\mathbf{d}$, near/far clipping planes ($t_{\text{near}}, t_{\text{far}}$), an occupancy grid structure, a coarse NeRF model, and a fine NeRF model [S1, unofficial].
*   **Outputs:** Rendered RGB color, depth, accumulated opacity, and weights for each ray [S1, unofficial].

**Data Structures:**
*   **OccupancyGrid:** A 3D boolean grid representing the spatial distribution of active volume data [S1, unofficial].
    *   `resolution`: Integer, the side length of the cubic grid.
    *   `aabb_min`: Tensor of shape `[3]`, minimum bounds of the axis-aligned bounding box (AABB) in world space.
    *   `aabb_max`: Tensor of shape `[3]`, maximum bounds of the AABB in world space.
    *   `density_threshold`: Float, the density threshold used to determine if a voxel is occupied.
    *   `grid`: Boolean tensor of shape `[resolution, resolution, resolution]`, where `True` indicates an occupied voxel.
    *   `voxel_size`: Tensor of shape `[3]`, the physical size of each voxel.

**Algorithm Steps:**

1.  **Normalize Ray Directions:**
    Normalize the input ray directions to unit length:
    $$ \mathbf{d}_{\text{norm}} = \frac{\mathbf{d}}{\|\mathbf{d}\|} $$ [S1, unofficial]

2.  **Coarse Sampling with Empty-Space Skipping:**
    Determine coarse sample positions $t_{\text{coarse}}$ along the ray [S1, unofficial].
    *   If an `OccupancyGrid` is available, query it to find occupied regions within $[t_{\text{near}}, t_{\text{far}}]$ and sample points within these segments [S1, unofficial].
    *   If no occupancy grid is present, perform standard stratified sampling across the entire interval $[t_{\text{near}}, t_{\text{far}}]$ [S1, unofficial].
    *   Let $N_c$ be the number of coarse samples (default 32) [S1, unofficial].

3.  **Coarse Rendering:**
    Evaluate the coarse model at positions $\mathbf{p}_i = \mathbf{o} + t_{\text{coarse}, i} \mathbf{d}_{\text{norm}}$ [S1, unofficial].
    *   Query the coarse model for density $\sigma_i$ and color $\mathbf{c}_i$ [S1, unofficial].
    *   Compute distances between adjacent samples: $d_i = t_{\text{coarse}, i+1} - t_{\text{coarse}, i}$ [S1, unofficial].
    *   Append a large distance (e.g., $10^{10}$) for the last segment to handle the background contribution [S1, unofficial].
    *   Compute alpha opacity:
        $$ \alpha_i = 1.0 - \exp(-\sigma_i \cdot d_i) $$ [S1, unofficial]
    *   Compute transmittance (accumulated opacity from previous samples):
        $$ T_i = \prod_{j=0}^{i-1} (1 - \alpha_j + \epsilon) $$
        where $\epsilon = 10^{-10}$ is added for numerical stability [S1, unofficial].
    *   Compute weights:
        $$ w_i = T_i \cdot \alpha_i $$ [S1, unofficial]
    *   Compute coarse color and depth:
        $$ \mathbf{C}_{\text{coarse}} = \sum_{i=0}^{N_c-1} w_i \mathbf{c}_i $$ [S1, unofficial]
        $$ D_{\text{coarse}} = \sum_{i=0}^{N_c-1} w_i t_{\text{coarse}, i} $$ [S1, unofficial]
        $$ A_{\text{coarse}} = \sum_{i=0}^{N_c-1} w_i $$ [S1, unofficial]

4.  **Hierarchical Fine Sampling:**
    Use the coarse weights $w_i$ to guide the placement of fine samples [S1, unofficial].
    *   Compute midpoints between coarse samples:
        $$ t_{\text{mid}, i} = \frac{t_{\text{coarse}, i} + t_{\text{coarse}, i+1}}{2} $$ [S1, unofficial]
    *   Construct bins for sampling:
        $$ \text{bins} = [t_{\text{coarse}, 0}, t_{\text{mid}, 0}, \dots, t_{\text{mid}, N_c-2}, t_{\text{coarse}, N_c-1}] $$ [S1, unofficial]
    *   Sample $N_f$ fine positions $t_{\text{fine}}$ from these bins, where the probability of sampling in a bin is proportional to the sum of weights of the coarse samples defining that bin [S1, unofficial]. This is done via inverse transform sampling using the cumulative weights [G].

5.  **Combine and Sort:**
    Concatenate coarse and fine samples:
    $$ t_{\text{combined}} = \text{sort}(t_{\text{coarse}} \cup t_{\text{fine}}) $$ [S1, unofficial]
    Let $N_{\text{total}} = N_c + N_f$.

6.  **Fine Rendering:**
    Evaluate the fine model at all positions in $t_{\text{combined}}$ [S1, unofficial].
    *   Query the fine model for density $\sigma'_i$ and color $\mathbf{c}'_i$ [S1, unofficial].
    *   Recompute $d_i, \alpha_i, T_i, w_i$ using the fine model's densities and the sorted $t_{\text{combined}}$ intervals [S1, unofficial].
    *   Compute final color:
        $$ \mathbf{C}_{\text{final}} = \sum_{i=0}^{N_{\text{total}}-1} w_i \mathbf{c}'_i $$ [S1, unofficial]
    *   Compute final depth:
        $$ D_{\text{final}} = \sum_{i=0}^{N_{\text{total}}-1} w_i t_{\text{combined}, i} $$ [S1, unofficial]
    *   Compute final accumulated opacity:
        $$ A_{\text{final}} = \sum_{i=0}^{N_{\text{total}}-1} w_i $$ [S1, unofficial]

7.  **Background Compositing:**
    If a white background is assumed, add the background contribution:
    $$ \mathbf{C}_{\text{out}} = \mathbf{C}_{\text{final}} + (1.0 - A_{\text{final}}) \cdot \mathbf{1} $$
    Otherwise, $\mathbf{C}_{\text{out}} = \mathbf{C}_{\text{final}}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def hierarchical_sampling_with_occupancy_grid(
    rays_o,       # [num_rays, 3]
    rays_d,       # [num_rays, 3]
    near,         # float
    far,          # float
    num_coarse,   # int
    num_fine,     # int
    occupancy_grid, # Optional OccupancyGrid object
    model_coarse, # Coarse NeRF model
    model_fine,   # Fine NeRF model
    white_background=True
):
    # 1. Normalize rays
    rays_d = torch.nn.functional.normalize(rays_d, dim=-1)
    
    # 2. Coarse Sampling
    if occupancy_grid is not None:
        occupied_intervals = occupancy_grid.sample_occupied_regions(rays_o, rays_d, near, far)
        t_coarse = sample_from_intervals(occupied_intervals, num_coarse)
    else:
        t_coarse = sample_stratified(near, far, num_coarse)
        
    # 3. Coarse Rendering
    positions_coarse = rays_o.unsqueeze(1) + t_coarse.unsqueeze(2) * rays_d.unsqueeze(1)
    pos_flat = positions_coarse.reshape(-1, 3)
    dir_flat = rays_d.unsqueeze(1).expand(-1, num_coarse, -1).reshape(-1, 3)
    
    rgb_coarse, density_coarse = model_coarse(pos_flat, dir_flat)
    rgb_coarse = rgb_coarse.reshape(-1, num_coarse, 3)
    density_coarse = density_coarse.reshape(-1, num_coarse)
    
    # Compute distances
    dists = t_coarse[:, 1:] - t_coarse[:, :-1]
    # Append a large distance for the last segment to handle background
    dists = torch.cat([dists, torch.full((dists.shape[0], 1), 1e10, device=dists.device)], dim=-1)
    
    # Compute Alpha
    alpha = 1.0 - torch.exp(-density_coarse * dists)
    
    # Compute Transmittance: T[i] = prod_{j=0}^{i-1} (1 - alpha[j] + eps)
    one_minus_alpha = 1.0 - alpha + 1e-10
    T = torch.cat([torch.ones_like(alpha[:, :1]), one_minus_alpha], dim=-1).cumprod(dim=-1)[:, :-1]
    
    weights_coarse = T * alpha
    
    # 4. Hierarchical Fine Sampling
    t_mid = 0.5 * (t_coarse[:, :-1] + t_coarse[:, 1:])
    bins = torch.cat([t_coarse[:, :1], t_mid, t_coarse[:, -1:]], dim=-1)
    
    t_fine = sample_hierarchical(bins, weights_coarse.detach(), num_fine)
    
    # 5. Combine and Sort
    t_combined = torch.cat([t_coarse, t_fine], dim=-1)
    t_combined, _ = torch.sort(t_combined, dim=-1)
    
    # 6. Fine Rendering
    positions_fine = rays_o.unsqueeze(1) + t_combined.unsqueeze(2) * rays_d.unsqueeze(1)
    pos_flat_fine = positions_fine.reshape(-1, 3)
    dir_flat_fine = rays_d.unsqueeze(1).expand(-1, t_combined.shape[1], -1).reshape(-1, 3)
    
    rgb_fine, density_fine = model_fine(pos_flat_fine, dir_flat_fine)
    rgb_fine = rgb_fine.reshape(-1, t_combined.shape[1], 3)
    density_fine = density_fine.reshape(-1, t_combined.shape[1])
    
    # Recompute distances for combined samples
    dists_combined = t_combined[:, 1:] - t_combined[:, :-1]
    dists_combined = torch.cat([dists_combined, torch.full((dists_combined.shape[0], 1), 1e10, device=dists_combined.device)], dim=-1)
    
    alpha_fine = 1.0 - torch.exp(-density_fine * dists_combined)
    one_minus_alpha_fine = 1.0 - alpha_fine + 1e-10
    T_fine = torch.cat([torch.ones_like(alpha_fine[:, :1]), one_minus_alpha_fine], dim=-1).cumprod(dim=-1)[:, :-1]
    
    weights_fine = T_fine * alpha_fine
    
    # 7. Composite
    rgb_integrated = torch.sum(weights_fine.unsqueeze(-1) * rgb_fine, dim=1)
    acc = torch.sum(weights_fine, dim=-1)
    depth = torch.sum(weights_fine * t_combined, dim=-1)
    
    if white_background:
        rgb_final = rgb_integrated + (1.0 - acc.unsqueeze(-1))
    else:
        rgb_final = rgb_integrated
        
    return rgb_final, depth, acc, weights_fine

def sample_stratified(near, far, num_samples):
    """Standard stratified sampling over [near, far]."""
    step = (far - near) / num_samples
    offsets = torch.rand_like(near) * step
    return near + torch.arange(num_samples, device=near.device).float() * step + offsets

def sample_from_intervals(intervals, num_samples):
    """Sample uniformly from a list of [start, end] intervals per ray."""
    # intervals: list of [num_rays, K] tensors
    # Flatten intervals to sample from
    starts = torch.stack([iv[:, 0] for iv in intervals], dim=-1)
    ends = torch.stack([iv[:, 1] for iv in intervals], dim=-1)
    lengths = ends - starts
    total_length = lengths.sum(dim=-1, keepdim=True)
    probs = lengths / (total_length + 1e-10)
    
    # Select interval index per sample
    u = torch.rand(num_samples, device=starts.device)
    cum_probs = torch.cumsum(probs, dim=-1)
    # Broadcast cum_probs to num_samples
    cum_probs = cum_probs.unsqueeze(0).expand(num_samples, -1, -1)
    u = u.unsqueeze(-1).unsqueeze(-1).expand(-1, cum_probs.shape[1], cum_probs.shape[2])
    bin_indices = torch.searchsorted(cum_probs, u, right=True)
    bin_indices = torch.clamp(bin_indices, 0, probs.shape[1] - 1)
    
    # Sample uniformly within selected interval
    local_u = torch.rand(num_samples, device=starts.device)
    selected_starts = torch.gather(starts, 1, bin_indices)
    selected_lengths = torch.gather(lengths, 1, bin_indices)
    return selected_starts + local_u * selected_lengths

def sample_hierarchical(bins, weights, num_samples):
    """Inverse transform sampling from bins weighted by coarse weights."""
    # Normalize weights to probabilities
    prob = weights / (weights.sum(dim=-1, keepdim=True) + 1e-10)
    # Cumulative distribution
    cdf = prob.cumsum(dim=-1)
    cdf = torch.cat([torch.zeros_like(cdf[:, :1]), cdf], dim=-1)
    
    # Generate uniform random numbers
    u = torch.rand_like(cdf)
    
    # Find bin index for each u
    bin_indices = torch.searchsorted(cdf, u, right=True) - 1
    bin_indices = torch.clamp(bin_indices, 0, bins.shape[1] - 2)
    
    # Interpolate within the bin
    bin_start = bins[torch.arange(bins.shape[0]), bin_indices]
    bin_end = bins[torch.arange(bins.shape[0]), bin_indices + 1]
    
    # Map u to the specific bin's local probability
    bin_cdf_start = cdf[torch.arange(cdf.shape[0]), bin_indices]
    bin_cdf_end = cdf[torch.arange(cdf.shape[0]), bin_indices + 1]
    local_u = (u - bin_cdf_start) / (bin_cdf_end - bin_cdf_start + 1e-10)
    
    return bin_start + local_u * (bin_end - bin_start)
```

**Parameters:**
*   `near`: Float, default 2.0. Near clipping plane distance [S1, unofficial].
*   `far`: Float, default 6.0. Far clipping plane distance [S1, unofficial].
*   `num_coarse_samples`: Integer, default 32. Number of samples taken along each ray in the coarse stage [S1, unofficial].
*   `num_fine_samples`: Integer, default 64. Number of samples taken along each ray in the fine stage [S1, unofficial].
*   `occupancy_grid`: Optional `OccupancyGrid` instance. If present, used for empty-space skipping during coarse sampling [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero in Ray-AABB Intersection:** When computing ray-box intersections for the occupancy grid, add a small epsilon ($10^{-8}$) to ray directions to prevent division by zero if a ray is parallel to a box face [S1, unofficial].
*   **Numerical Stability in Transmittance:** Add $10^{-10}$ to $(1 - \alpha)$ before computing the cumulative product to prevent `NaN` values due to floating-point underflow [S1, unofficial].
*   **Model Input Compatibility:** If the model does not support `forward_with_density_only`, dummy zero directions may need to be passed to satisfy input shape requirements during density-only queries [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** FLASH achieves training speeds of 10-15 minutes on a single GPU for synthetic scenes, significantly faster than vanilla NeRF [S1, unofficial].
*   **Strengths:** Efficiently skips empty space, reducing the number of model evaluations. Hierarchical sampling focuses computational effort on high-density regions, improving image quality per sample [S1, unofficial].
*   **Limitations:** Requires construction and maintenance of the occupancy grid. Performance depends on the sparsity of the scene; if the scene is dense, the occupancy grid provides less benefit [S1, unofficial].

## Background
Hierarchical Sampling with Occupancy Grid is implemented in the FLASH (Fast Learning for Accurate Scene Hashing) project, an experimental implementation of Instant-NGP for Neural Radiance Fields [S1, unofficial]. It was developed to enable rapid training of 3D scenes from 2D images, leveraging multi-resolution hash encoding and occupancy grids for acceleration [S1, unofficial].

## Sources
[S1] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
