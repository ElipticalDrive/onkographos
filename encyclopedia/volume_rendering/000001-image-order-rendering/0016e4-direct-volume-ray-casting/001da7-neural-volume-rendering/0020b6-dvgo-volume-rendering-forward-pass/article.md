# DVGO Volume Rendering Forward Pass

Also called: DVGO forward, Direct Voxel Grid Optimization rendering

## Overview
The DVGO forward pass renders a 3D scene represented by a direct voxel grid of density and color features into a 2D image by querying the grid along camera rays and compositing the results [S1, unofficial]. It differs from standard NeRF by replacing the multi-layer perceptron (MLP) with a direct lookup in a 3D texture (voxel grid), enabling significantly faster training and inference while maintaining view-dependent color through auxiliary feature vectors [S1, S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   `rays_o`: Origin of each ray, shape $(N, 3)$, where $N$ is the number of pixels (rays) [S1, unofficial].
*   `rays_d`: Direction of each ray, shape $(N, 3)$ [S1, unofficial].
*   `near`, `far`: Near and far clipping planes for ray marching [S1, unofficial].
*   `stepsize`: Distance between sample points along the ray [S1, unofficial].
*   `render_kwargs`: Dictionary containing optional flags (`render_features`, `render_diffuse`, `render_depth`) and background color `bg` [S1, unofficial].

**Outputs:**
*   `rgb_marched`: Final rendered RGB image, shape $(N, 3)$ [S1, unofficial].
*   `alphainv_last`: Accumulated transmittance at the end of the ray, shape $(N)$ [S1, unofficial].
*   `weights`: Per-sample weights used for compositing, shape $(M)$, where $M$ is the number of valid samples [S1, unofficial].
*   `raw_alpha`: Raw alpha values before weighting, shape $(M)$ [S1, unofficial].
*   `raw_rgb`: Raw RGB values before weighting, shape $(M, 3)$ [S1, unofficial].
*   `ray_id`: Mapping from sample index to ray index, shape $(M)$ [S1, unofficial].
*   `depth` (optional): Weighted average depth, shape $(N)$ [S1, unofficial].
*   `diffuse_marched` (optional): Rendered diffuse color, shape $(N, 3)$ [S1, unofficial].

### Data Structures
*   **Density Grid ($V_\sigma$)**: A 3D tensor of shape $(D_x, D_y, D_z)$ storing raw density values [S1, unofficial].
*   **Color Feature Grid ($V_c$)**: A 3D tensor of shape $(D_x, D_y, D_z, k_0)$ storing feature vectors. $k_0$ is the feature dimension [S1, unofficial].
*   **RGB Network ($\text{rgbnet}$)**: A small MLP (optional) that maps color features and view direction to final RGB [S1, unofficial].
*   **Diffuse Color Grid ($V_d$)**: A 3D tensor of shape $(D_x, D_y, D_z, 3)$ storing diffuse color components (if `use_diffuse_color` is enabled) [S1, unofficial].
*   **Mask Cache**: A boolean grid or hash map indicating free space, used to skip empty regions [S1, unofficial].

### Parameters
*   `interval`: Sampling interval for density-to-alpha conversion. Calculated as `stepsize * voxel_size_ratio` [S1, unofficial].
*   `act_shift`: Shift parameter for the density activation function (e.g., sigmoid shift) [S1, unofficial].
*   `fast_color_thres`: Threshold for early ray termination. If alpha or weight exceeds this, subsequent samples are skipped [S1, unofficial].
*   `use_diffuse_color`: Boolean flag to enable diffuse color rendering [S1, unofficial].
*   `rgbnet_full_implicit`: Boolean flag determining if view direction is concatenated with features before the RGB network [S1, unofficial].
*   `rgbnet_direct`: Boolean flag determining if diffuse color is added directly to the network output [S1, unofficial].

### Steps

1.  **Ray Sampling**: Generate sample points along each ray. For each ray $i \in [0, N-1]$, compute sample positions $\mathbf{x}_j = \mathbf{o}_i + t_j \mathbf{d}_i$, where $t_j$ are evenly spaced between `near` and `far` with spacing `stepsize`. Collect all points into `ray_pts` of shape $(M, 3)$, along with corresponding `ray_id` and `step_id` (index along the ray) [S1, unofficial].
2.  **Free Space Culling**: If `mask_cache` is present, filter `ray_pts`, `ray_id`, and `step_id` to keep only points where `mask_cache(ray_pts)` is true [S1, unofficial].
3.  **Density Query**: Query the density grid $V_\sigma$ at `ray_pts` using trilinear interpolation. Let $\sigma_j$ be the raw density value for sample $j$ [S1, unofficial].
4.  **Density to Alpha Conversion**: Convert raw density $\sigma_j$ to alpha $\alpha_j$ using the `Raw2Alpha` function: $\alpha_j = 1 - \exp(-(\sigma_j + \text{act\_shift}) \cdot \text{interval})$ [S1, unofficial].
5.  **Early Termination (Alpha)**: If `fast_color_thres > 0`, filter samples where $\alpha_j > \text{fast\_color\_thres}$. Remove these samples from `ray_pts`, `ray_id`, `step_id`, `density`, and `alpha` [S1, unofficial].
6.  **Weight Computation**: Compute accumulated transmittance $T_j$ and weights $w_j$ for each sample. $T_1 = 1$, $T_{j} = T_{j-1} \cdot (1 - \alpha_{j-1})$ for $j > 1$ (assuming sorted by depth). $w_j = T_j \cdot \alpha_j$. Compute `alphainv_last` (total transmittance at the end of the ray): $T_{M+1} = T_M \cdot (1 - \alpha_M)$ [S1, unofficial].
7.  **Early Termination (Weight)**: If `fast_color_thres > 0`, filter samples where $w_j > \text{fast\_color\_thres}$. Remove these samples [S1, unofficial].
8.  **Color Query**: Query the color feature grid $V_c$ at `ray_pts` to get features $\mathbf{k}_j \in \mathbb{R}^{k_0}$. If `use_diffuse_color` is true, query the diffuse color grid $V_d$ to get $\mathbf{d}_j \in \mathbb{R}^3$ [S1, unofficial].
9.  **RGB Computation**: If `rgbnet` is None, set $\mathbf{c}_j = \text{sigmoid}(\mathbf{d}_j)$ (if diffuse is used) or use features directly [S1, unofficial]. If `rgbnet` is not None, compute positional encoding of view direction $\mathbf{d}_i$ for the ray containing sample $j$, concatenate with $\mathbf{k}_j$, pass through `rgbnet`, and apply sigmoid. If `rgbnet_direct` is true, add diffuse color: $\mathbf{c}_j = \mathbf{c}_j + \mathbf{d}_j$ [S1, unofficial].
10. **Compositing**: Compute final RGB for each ray $i$: $\text{rgb}_i = \sum_{j \in \text{ray } i} w_j \cdot \mathbf{c}_j + T_{M+1} \cdot \mathbf{bg}$, where $\mathbf{bg}$ is the background color [S1, unofficial].
11. **Optional Outputs**: If `render_features`: Sum `weights * k_j` grouped by `ray_id`. If `render_diffuse`: Sum `weights * sigmoid(d_j)` grouped by `ray_id`. If `render_depth`: Sum `weights * step_id` grouped by `ray_id` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def dvgo_forward_pass(rays_o, rays_d, near, far, stepsize, model, render_kwargs):
    N = rays_o.shape[0]
    
    # 1. Ray Sampling
    num_samples = int((far - near) / stepsize)
    t_vals = torch.linspace(0., 1., num_samples, device=rays_o.device)
    t_vals = near + t_vals * (far - near)
    ray_pts = rays_o.unsqueeze(1) + rays_d.unsqueeze(1) * t_vals.unsqueeze(2)
    M = N * num_samples
    ray_pts_flat = ray_pts.reshape(-1, 3)
    ray_id = torch.arange(N, device=rays_o.device).unsqueeze(1).expand(-1, num_samples).reshape(-1)
    step_id = torch.arange(num_samples, device=rays_o.device).unsqueeze(0).expand(N, -1).reshape(-1)
    
    # 2. Free Space Culling
    if model.mask_cache is not None:
        valid_mask = model.mask_cache(ray_pts_flat)
        ray_pts_flat = ray_pts_flat[valid_mask]
        ray_id = ray_id[valid_mask]
        step_id = step_id[valid_mask]
        M = ray_pts_flat.shape[0]
        if M == 0:
            return {
                'rgb_marched': torch.zeros(N, 3, device=rays_o.device) + render_kwargs['bg'],
                'alphainv_last': torch.ones(N, device=rays_o.device),
                'weights': torch.zeros(0, device=rays_o.device),
                'raw_alpha': torch.zeros(0, device=rays_o.device),
                'raw_rgb': torch.zeros(0, 3, device=rays_o.device),
                'ray_id': torch.zeros(0, dtype=torch.long, device=rays_o.device)
            }

    # 3. Density Query
    norm_pts = (ray_pts_flat - model.xyz_min) / (model.xyz_max - model.xyz_min)
    norm_pts = torch.clamp(norm_pts, 0.0, 1.0)
    density = F.grid_sample(
        model.density_grid.unsqueeze(0).unsqueeze(0),
        norm_pts.unsqueeze(0).unsqueeze(0).unsqueeze(0),
        align_corners=True
    ).squeeze()
    
    # 4. Density to Alpha
    interval = stepsize * model.voxel_size_ratio
    alpha = 1.0 - torch.exp(-(density + model.act_shift) * interval)
    
    # 5. Early Termination (Alpha)
    if model.fast_color_thres > 0:
        keep_mask = alpha > model.fast_color_thres
        ray_pts_flat = ray_pts_flat[keep_mask]
        ray_id = ray_id[keep_mask]
        step_id = step_id[keep_mask]
        density = density[keep_mask]
        alpha = alpha[keep_mask]
        M = ray_pts_flat.shape[0]

    # 6. Weight Computation
    # Sort by ray_id and step_id to ensure correct order per ray
    sort_idx = torch.argsort(ray_id * 100000 + step_id)
    ray_id = ray_id[sort_idx]
    step_id = step_id[sort_idx]
    alpha = alpha[sort_idx]
    
    # Compute transmittance T per ray
    T = torch.ones(M, device=rays_o.device)
    one_minus_alpha = 1.0 - alpha
    cum_prod = torch.cumprod(one_minus_alpha, dim=0)
    # T[j] = 1 if j is start of ray, else cum_prod[j-1]
    ray_starts = torch.zeros(N, dtype=torch.long, device=rays_o.device)
    counts = torch.bincount(ray_id, minlength=N)
    ray_starts = torch.cat([torch.tensor([0], device=rays_o.device), torch.cumsum(counts[:-1], dim=0)])
    T[ray_starts] = 1.0
    T[ray_starts[1:]] = cum_prod[ray_starts[1:] - 1]
    
    weights = T * alpha
    # alphainv_last per ray is T at the last sample * (1 - alpha at last sample)
    last_indices = ray_starts + counts - 1
    alphainv_last_per_ray = T[last_indices] * (1 - alpha[last_indices])
    
    # 7. Early Termination (Weight)
    if model.fast_color_thres > 0:
        weight_mask = weights > model.fast_color_thres
        ray_pts_flat = ray_pts_flat[weight_mask]
        ray_id = ray_id[weight_mask]
        step_id = step_id[weight_mask]
        alpha = alpha[weight_mask]
        weights = weights[weight_mask]
        T = T[weight_mask]
        M = ray_pts_flat.shape[0]
        
        counts = torch.bincount(ray_id, minlength=N)
        ray_starts = torch.cat([torch.tensor([0], device=ray_id.device), torch.cumsum(counts[:-1], dim=0)])
        last_indices = ray_starts + counts - 1
        alphainv_last_per_ray = T[last_indices] * (1 - alpha[last_indices])

    # 8. Color Query
    norm_pts_filtered = (ray_pts_flat - model.xyz_min) / (model.xyz_max - model.xyz_min)
    norm_pts_filtered = torch.clamp(norm_pts_filtered, 0.0, 1.0)
    
    color_features = F.grid_sample(
        model.color_grid.unsqueeze(0).unsqueeze(0),
        norm_pts_filtered.unsqueeze(0).unsqueeze(0).unsqueeze(0),
        align_corners=True
    ).squeeze()
    
    rgb = torch.zeros(M, 3, device=rays_o.device)
    diffuse = torch.zeros(M, 3, device=rays_o.device)
    
    if model.use_diffuse_color:
        diffuse = F.grid_sample(
            model.diffuse_grid.unsqueeze(0).unsqueeze(0),
            norm_pts_filtered.unsqueeze(0).unsqueeze(0).unsqueeze(0),
            align_corners=True
        ).squeeze()
        
    if model.rgbnet is None:
        if model.use_diffuse_color:
            rgb = torch.sigmoid(diffuse)
        else:
            if color_features.shape[1] == 3:
                rgb = color_features
            else:
                rgb = torch.zeros(M, 3, device=rays_o.device)
    else:
        viewdirs = rays_d[ray_id]
        viewdirs_pe = positional_encoding(viewdirs)
        input_color = torch.cat([color_features, viewdirs_pe], dim=-1)
        raw_rgb = model.rgbnet(input_color)
        rgb = torch.sigmoid(raw_rgb)
        if model.rgbnet_direct and model.use_diffuse_color:
            rgb = rgb + torch.sigmoid(diffuse)

    # 9. Compositing
    rgb_marched = segment_coo(
        src=weights.unsqueeze(-1) * rgb,
        index=ray_id,
        out=torch.zeros(N, 3, device=rays_o.device),
        reduce='sum'
    )
    bg = render_kwargs['bg'].to(rays_o.device)
    rgb_marched = rgb_marched + alphainv_last_per_ray.unsqueeze(-1) * bg
    
    # 10. Optional Outputs
    extras = {}
    if render_kwargs.get('render_features', False):
        extras['features_marched'] = segment_coo(
            src=weights.unsqueeze(-1) * color_features,
            index=ray_id,
            out=torch.zeros(N, color_features.shape[1], device=rays_o.device),
            reduce='sum'
        )
    if model.use_diffuse_color and render_kwargs.get('render_diffuse', False):
        extras['diffuse_marched'] = segment_coo(
            src=weights.unsqueeze(-1) * torch.sigmoid(diffuse),
            index=ray_id,
            out=torch.zeros(N, 3, device=rays_o.device),
            reduce='sum'
        )
    if render_kwargs.get('render_depth', False):
        extras['depth'] = segment_coo(
            src=weights.unsqueeze(-1) * step_id.unsqueeze(-1).float(),
            index=ray_id,
            out=torch.zeros(N, 1, device=rays_o.device),
            reduce='sum'
        ).squeeze(-1)

    return {
        'rgb_marched': rgb_marched,
        'alphainv_last': alphainv_last_per_ray,
        'weights': weights,
        'raw_alpha': alpha,
        'raw_rgb': rgb,
        'ray_id': ray_id,
        **extras
    }

def positional_encoding(x, min_deg=0, max_deg=10, scale=1, base_freq=2.0):
    """Standard sinusoidal positional encoding [G]."""
    num_bands = max_deg - min_deg
    freq_bands = scale * (2.0 ** torch.arange(min_deg, max_deg, device=x.device).float())
    scaled_x = x.unsqueeze(-1) * freq_bands.unsqueeze(0)
    encoded = torch.cat([torch.sin(scaled_x), torch.cos(scaled_x)], dim=-1)
    return encoded.reshape(x.shape[0], -1)

def segment_coo(src, index, out, reduce='sum'):
    """PyTorch segment reduction using scatter_add_."""
    if reduce == 'sum':
        out = out.expand_as(src)
        out.scatter_add_(0, index.unsqueeze(-1).expand_as(out), src)
        return out
    else:
        raise NotImplementedError("Only 'sum' reduction is supported.")
```

### Edge Cases and Pitfalls
*   **Zero Density Initialization**: DVGO initializes the density grid with zeros, resulting in nearly zero entropy initially. This prevents guiding the agent to choose the next-best view with highest information gain in active vision tasks [S2, unofficial].
*   **Precision**: Trilinear interpolation in the voxel grid can introduce artifacts if the grid resolution is too low.
*   **Boundaries**: Ray points outside the voxel grid bounds must be clamped or filtered to avoid out-of-bounds memory access.
*   **Early Termination Threshold**: Setting `fast_color_thres` too high may skip valid samples, leading to inaccurate rendering. Setting it too low may not provide performance benefits.

## Subtopics

### Entropy Volume Initialization
An additional entropy volume $V(E,c)$ introduced to overcome the zero-entropy initialization issue of DVGO's density volume, allowing information gain calculation at the start. It stores $\alpha(E,c)$, initialized with 0.5 to represent highest entropy. For voxels of $V(\sigma,c)$ that have been updated in the reconstruction stage, $\alpha(E,c)$ is synchronized with $\alpha(\sigma,c)$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot S)$, where $N$ is the number of pixels and $S$ is the number of samples per ray. The lookup is $O(1)$ per sample, making it faster than MLP-based methods which are $O(N \cdot S \cdot D)$ where $D$ is the MLP depth.
*   **Memory Complexity**: $O(D_x \cdot D_y \cdot D_z \cdot (1 + k_0 + 3))$, where $D_x, D_y, D_z$ are the grid dimensions, $k_0$ is the color feature dimension, and 3 is for diffuse color. This is significantly less than storing a full radiance field MLP weights for high-resolution scenes.
*   **Performance**: Faster training and inference compared to NeRF due to the direct voxel grid lookup. However, it may suffer from aliasing artifacts if the grid resolution is not high enough.
*   **Strengths**: Fast training, efficient memory usage for high-resolution scenes, easy to integrate with existing voxel-based pipelines.
*   **Limitations**: Lower quality than NeRF in complex scenes due to the discrete nature of the voxel grid. View-dependent effects are limited by the feature dimension $k_0$.

## Background
DVGO (Direct Voxel Grid Optimization) was introduced as a method to accelerate NeRF training by replacing the MLP with a direct voxel grid. It builds on the principles of neural volume rendering [S1, S2, unofficial]. The method was developed to address the computational bottleneck of MLP queries in NeRF, enabling real-time or near-real-time training and rendering.

## Sources
[S1] gkouros/ref-dvgo, https://github.com/gkouros/ref-dvgo, fetched 2026-10-28 (unofficial)
[S2] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
