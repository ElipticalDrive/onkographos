# Instant-NSR-PL Forward Pass

Also called: Instant-NSR-PL rendering, Neural SDF rendering with finite difference gradients

## Overview
The Instant-NSR-PL Forward Pass renders scenes by marching rays through a neural Signed Distance Function (SDF) and accumulating color, normals, depth, and opacity along each ray [S1, unofficial]. It computes the composite RGB image, surface normals, depth, and opacity by querying a geometry network for SDF values and gradients, computing opacity from the SDF and ray direction, and accumulating results along the ray [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- `rays_o`: Tensor of ray origins, shape $(N, 3)$ [S1, unofficial].
- `rays_d`: Tensor of ray directions, shape $(N, 3)$ [S1, unofficial].
- `t_starts`: Tensor of near clipping distances for each ray, shape $(N)$ [S1, unofficial].
- `t_ends`: Tensor of far clipping distances for each ray, shape $(N)$ [S1, unofficial].
- `ray_indices`: Tensor mapping sample indices to ray indices, used for accumulation [S1, unofficial].
- `n_rays`: Integer, total number of rays [S1, unofficial].
- `config`: Configuration object containing `geometry.grad_type` (e.g., 'finite_difference' or 'analytical'), `learned_background`, `background_color`, and `randomized` [S1, unofficial].

**Outputs:**
- Dictionary containing:
  - `comp_rgb`: Composite RGB color, shape $(N, 3)$ [S1, unofficial].
  - `comp_normal`: Composite surface normal, shape $(N, 3)$ [S1, unofficial].
  - `opacity`: Composite opacity, shape $(N)$ [S1, unofficial].
  - `depth`: Composite depth, shape $(N)$ [S1, unofficial].
  - `rays_valid`: Boolean mask indicating rays that hit geometry, shape $(N)$ [S1, unofficial].
  - `num_samples`: Number of samples per ray [S1, unofficial].
  - (Training only) Intermediate tensors: `sdf_samples`, `sdf_grad_samples`, `weights`, `points`, `intervals`, `ray_indices`, and optionally `sdf_laplace_samples` [S1, unofficial].

### Data Structures
- **Tensors**: All geometric and color data are stored in PyTorch tensors [S1, unofficial].
- **Sample Points**: Positions are computed as midpoints between `t_starts` and `t_ends` [S1, unofficial].
- **Accumulation Buffers**: Separate buffers for opacity, depth, RGB, and normal are maintained per ray index [S1, unofficial].

### Steps
1. **Extract Ray Data**: Retrieve origins `t_origins` and directions `t_dirs` for the current batch of rays using `ray_indices` [S1, unofficial].
2. **Compute Sample Positions**: Calculate segment distances $dists = t_{ends} - t_{starts}$ [S1, unofficial]. Calculate midpoints $midpoints = \frac{t_{starts} + t_{ends}}{2}$ [S1, unofficial]. Calculate world positions $positions = t_{origins} + t_{dirs} \times midpoints$ [S1, unofficial].
3. **Query Geometry Network**: If `config.geometry.grad_type` is 'finite_difference', query the geometry network with `with_grad=True`, `with_feature=True`, and `with_laplace=True` [S1, unofficial]. Otherwise, query with `with_grad=True` and `with_feature=True` to obtain analytical gradients [S1, unofficial]. Let $S$ be the SDF value, $G$ be the gradient, $F$ be the feature vector, and $L$ be the Laplacian (if requested) [S1, unofficial].
4. **Compute Normals**: Normalize the gradient to obtain surface normals: $N = \text{normalize}(G)$ [S1, unofficial].
5. **Compute Opacity**: Compute alpha (opacity) for each sample using the SDF value $S$, normal $N$, ray direction $D$, and segment distance $dists$ [S1, unofficial]. The specific formula for `get_alpha` is not detailed in the source [G]. $\alpha = \text{get\_alpha}(S, N, D, dists)$ [S1, unofficial].
6. **Compute Color**: Compute RGB color using the feature vector $F$, ray direction $D$, and normal $N$: $RGB = \text{texture}(F, D, N)$ [S1, unofficial].
7. **Compute Rendering Weights**: Compute weights from alpha values using `render_weight_from_alpha` [S1, unofficial].
8. **Accumulate Results**: Accumulate opacity: $Opacity = \sum w_i$ [S1, unofficial]. Accumulate depth: $Depth = \sum w_i \times midpoints_i$ [S1, unofficial]. Accumulate RGB: $CompRGB = \sum w_i \times RGB_i$ [S1, unofficial]. Accumulate normal: $CompNormal_{raw} = \sum w_i \times N_i$ [S1, unofficial]. Normalize the composite normal: $CompNormal = \text{normalize}(CompNormal_{raw})$ [S1, unofficial].
9. **Construct Output**: Create a validity mask: $Valid = Opacity > 0$ [S1, unofficial]. If training, store intermediate tensors for gradient computation [S1, unofficial].
10. **Handle Background**: If `config.learned_background` is true, call the background rendering function `forward_bg_` [S1, unofficial]. Otherwise, create a background output with RGB set to `config.background_color` and zeroed sample counts/validity [S1, unofficial]. Combine foreground and background: $FinalRGB = CompRGB + BackgroundRGB \times (1.0 - Opacity)$ [S1, unofficial]. Update `num_samples` and `rays_valid` by addition and logical OR, respectively [S1, unofficial].
11. **Return Output**: Return the combined output dictionary [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def forward_pass(rays_o, rays_d, t_starts, t_ends, ray_indices, n_rays, config, geometry_net, texture_net, background_color):
    # 1. Extract Ray Data
    t_origins = rays_o[ray_indices]
    t_dirs = rays_d[ray_indices]
    
    # 2. Compute Sample Positions
    dists = t_ends - t_starts
    midpoints = (t_starts + t_ends) / 2.0
    positions = t_origins + t_dirs * midpoints
    
    # 3. Query Geometry Network
    if config.geometry.grad_type == 'finite_difference':
        sdf, sdf_grad, feature, sdf_laplace = geometry_net(positions, with_grad=True, with_feature=True, with_laplace=True)
    else:
        sdf, sdf_grad, feature = geometry_net(positions, with_grad=True, with_feature=True)
        sdf_laplace = None
        
    # 4. Compute Normals
    normal = F.normalize(sdf_grad, p=2, dim=-1)
    
    # 5. Compute Opacity
    # Formula for get_alpha is not given in the sources [G]
    alpha = get_alpha(sdf, normal, t_dirs, dists)
    alpha = alpha[..., None]  # Shape: (..., 1)
    
    # 6. Compute Color
    rgb = texture_net(feature, t_dirs, normal)
    
    # 7. Compute Rendering Weights
    weights = render_weight_from_alpha(alpha, ray_indices=ray_indices, n_rays=n_rays)
    
    # 8. Accumulate Results
    opacity = accumulate_along_rays(weights, ray_indices, values=None, n_rays=n_rays)
    depth = accumulate_along_rays(weights, ray_indices, values=midpoints, n_rays=n_rays)
    comp_rgb = accumulate_along_rays(weights, ray_indices, values=rgb, n_rays=n_rays)
    
    comp_normal_raw = accumulate_along_rays(weights, ray_indices, values=normal, n_rays=n_rays)
    comp_normal = F.normalize(comp_normal_raw, p=2, dim=-1)
    
    # 9. Construct Output
    out = {
        'comp_rgb': comp_rgb,
        'comp_normal': comp_normal,
        'opacity': opacity,
        'depth': depth,
        'rays_valid': opacity > 0,
        'num_samples': torch.zeros(n_rays, dtype=torch.long, device=opacity.device)
    }
    
    # Training intermediates
    if config.training:
        out['sdf_samples'] = sdf
        out['sdf_grad_samples'] = sdf_grad
        out['weights'] = weights.view(-1)
        out['points'] = midpoints.view(-1)
        out['intervals'] = dists.view(-1)
        out['ray_indices'] = ray_indices.view(-1)
        if config.geometry.grad_type == 'finite_difference':
            out['sdf_laplace_samples'] = sdf_laplace
            
    # 10. Handle Background
    if config.learned_background:
        out_bg = forward_bg_(rays_o, rays_d, config)
    else:
        out_bg = {
            'comp_rgb': torch.full_like(comp_rgb, background_color),
            'opacity': torch.zeros_like(opacity),
            'rays_valid': torch.zeros_like(opacity, dtype=torch.bool),
            'num_samples': torch.zeros_like(opacity, dtype=torch.long)
        }
        
    # Combine foreground and background
    final_rgb = out['comp_rgb'] + out_bg['comp_rgb'] * (1.0 - out['opacity'])
    out['comp_rgb'] = final_rgb
    out['num_samples'] = out['num_samples'] + out_bg['num_samples']
    out['rays_valid'] = out['rays_valid'] | out_bg['rays_valid']
    
    return out

def get_alpha(sdf, normal, ray_dir, dists):
    # The specific formula for get_alpha is not given in the sources [G]
    # Standard differentiable volume rendering uses: alpha = sigmoid(sdf * sigma) * (1 - exp(-sigma * dists))
    sigma = 10.0  # Parameter value not given in the sources [G]
    return torch.sigmoid(sdf * sigma) * (1.0 - torch.exp(-sigma * dists))

def render_weight_from_alpha(alpha, ray_indices, n_rays):
    # Compute weights: w_i = alpha_i * prod_{j < i, same ray} (1 - alpha_j)
    # Sort samples by ray index to compute cumulative products correctly
    sorted_indices = torch.argsort(ray_indices)
    sorted_alpha = alpha[sorted_indices]
    ones_minus_alpha = 1.0 - sorted_alpha
    
    # Cumulative product of (1 - alpha) along each ray
    cum_prod = torch.cumprod(ones_minus_alpha, dim=0)
    # Shift cum_prod by 1 to exclude current sample's (1-alpha)
    cum_prod_shifted = torch.cat([torch.ones_like(cum_prod[:1]), cum_prod[:-1]], dim=0)
    
    weights = sorted_alpha * cum_prod_shifted
    # Restore original order
    weights[sorted_indices] = weights
    return weights

def accumulate_along_rays(weights, ray_indices, values, n_rays):
    # Initialize accumulators
    if values is None:
        accum = torch.zeros(n_rays, device=weights.device, dtype=weights.dtype)
    else:
        accum = torch.zeros(n_rays, *values.shape[1:], device=values.device, dtype=values.dtype)
    
    # Scatter add weights * values to corresponding ray indices
    if values is None:
        accum.scatter_add_(0, ray_indices, weights)
    else:
        weights_expanded = weights.unsqueeze(-1).expand_as(values)
        accum.scatter_add_(0, ray_indices.unsqueeze(-1).expand_as(values), weights_expanded * values)
    return accum

def forward_bg_(rays_o, rays_d, config):
    # Background rendering logic is external to the forward pass core [G]
    pass
```

### Parameters
- `config.geometry.grad_type`: String, 'finite_difference' or 'analytical'. Determines how SDF gradients are computed [S1, unofficial].
- `config.learned_background`: Boolean, whether to use a learned background model [S1, unofficial].
- `config.background_color`: Tensor, color used for background if not learned [S1, unofficial].
- `config.ray_chunk`: Integer, batch size for ray processing during evaluation to manage memory [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision**: Finite difference gradient computation can be sensitive to step size [S1, unofficial].
- **Boundaries**: Ray origins and directions must be correctly aligned with the coordinate system used by the geometry network [S1, unofficial].
- **Degenerate Input**: Rays with `t_starts == t_ends` should be handled to avoid invalid segment distances [S1, unofficial].
- **Background Handling**: If `learned_background` is false, the background color must be consistent with the training data to avoid artifacts [S1, unofficial].

## Subtopics

### Differentiable Silhouette Rendering
Renders binary silhouettes using SDFs. Choose when only shape boundary is needed, not color or normals. [Link](4322-differentiable-silhouette-rendering.md)

### SDF Direct Diffuse Integrator
Integrates diffuse shading directly from SDFs. Choose for physically based diffuse rendering without learned texture networks. [Link](4467-sdf-direct-diffuse-integrator.md)

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of samples per ray and the number of rays [S1, unofficial].
- **Memory Complexity**: Linear with respect to the number of samples [S1, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales with the number of rays and samples. Chunking is used during evaluation to manage memory [S1, unofficial].
- **Strengths**: Differentiable, supports gradient computation with respect to scene parameters, handles complex geometry via SDFs [S1, unofficial].
- **Limitations**: Requires careful tuning of SDF query parameters, finite difference gradients can be noisy [S1, unofficial].
- **When to Use**: When differentiable rendering of complex 3D scenes with learned textures is required [S1, unofficial].
- **When Not to Use**: When real-time performance is critical and differentiability is not needed, or when analytical gradients are preferred over finite differences [S1, unofficial].

## Background
The Instant-NSR-PL Forward Pass is part of the Instant-Neural SDF Rendering (Instant-NSR) method, introduced by Guo et al. in the repository `bennyguo/instant-nsr-pl` [S1, unofficial]. It builds on the principles of Neural SDFs and differentiable rendering, aiming to provide a fast and flexible framework for rendering 3D scenes from 2D images [S1, unofficial]. The method supports both analytical and finite-difference gradients for the SDF, allowing for flexibility in training and inference [S1, unofficial].

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
