# Instant-NSR-PL Forward Pass

Also called: Instant-NSR-PL rendering, Neural SDF rendering with finite difference gradients

## Overview
The Instant-NSR-PL Forward Pass is the core rendering kernel for the Instant-Neural SDF Rendering (Instant-NSR) method, which renders scenes using differentiable sphere tracing. It computes the composite RGB image, surface normals, depth, and opacity by marching along rays, querying a neural Signed Distance Function (SDF) network, and accumulating color based on opacity derived from the SDF value and surface orientation. It is distinguished from its sibling [Differentiable Silhouette Rendering](4322-differentiable-silhouette-rendering.md) by its volumetric accumulation of color and normals rather than binary silhouette masks, and from the [SDF Direct Diffuse Integrator](4467-sdf-direct-diffuse-integrator.md) by its use of a learned texture network and finite-difference or analytical gradients for the SDF rather than direct analytical integration of diffuse shading.

## How it works
The method processes a batch of rays, querying a geometry network for SDF values and gradients, computing opacity from the SDF and ray direction, and accumulating color and normals along the ray.

### Inputs and Outputs
**Inputs:**
- `rays_o`: Tensor of ray origins, shape $(N, 3)$.
- `rays_d`: Tensor of ray directions, shape $(N, 3)$.
- `t_starts`: Tensor of near clipping distances for each ray, shape $(N)$.
- `t_ends`: Tensor of far clipping distances for each ray, shape $(N)$.
- `ray_indices`: Tensor mapping sample indices to ray indices, used for accumulation.
- `n_rays`: Integer, total number of rays.
- `config`: Configuration object containing `geometry.grad_type` (e.g., 'finite_difference' or 'analytical'), `learned_background`, `background_color`, and `randomized`.

**Outputs:**
- Dictionary containing:
  - `comp_rgb`: Composite RGB color, shape $(N, 3)$.
  - `comp_normal`: Composite surface normal, shape $(N, 3)$.
  - `opacity`: Composite opacity, shape $(N)$.
  - `depth`: Composite depth, shape $(N)$.
  - `rays_valid`: Boolean mask indicating rays that hit geometry, shape $(N)$.
  - `num_samples`: Number of samples per ray.
  - (Training only) Intermediate tensors: `sdf_samples`, `sdf_grad_samples`, `weights`, `points`, `intervals`, `ray_indices`, and optionally `sdf_laplace_samples`.

### Data Structures
- **Tensors**: All geometric and color data are stored in PyTorch tensors.
- **Sample Points**: Positions are computed as midpoints between `t_starts` and `t_ends`.
- **Accumulation Buffers**: Separate buffers for opacity, depth, RGB, and normal are maintained per ray index.

### Steps
1. **Extract Ray Data**: Retrieve origins `t_origins` and directions `t_dirs` for the current batch of rays using `ray_indices`.
2. **Compute Sample Positions**:
   - Calculate segment distances: $dists = t_{ends} - t_{starts}$.
   - Calculate midpoints: $midpoints = \frac{t_{starts} + t_{ends}}{2}$.
   - Calculate world positions: $positions = t_{origins} + t_{dirs} \times midpoints$.
3. **Query Geometry Network**:
   - If `config.geometry.grad_type` is 'finite_difference', query the geometry network with `with_grad=True`, `with_feature=True`, and `with_laplace=True`. This typically involves perturbing positions to estimate gradients via finite differences.
   - Otherwise, query with `with_grad=True` and `with_feature=True` to obtain analytical gradients.
   - Let $S$ be the SDF value, $G$ be the gradient, $F$ be the feature vector, and $L$ be the Laplacian (if requested).
4. **Compute Normals**:
   - Normalize the gradient to obtain surface normals: $N = \text{normalize}(G)$.
5. **Compute Opacity**:
   - Compute alpha (opacity) for each sample using the SDF value $S$, normal $N$, ray direction $D$, and segment distance $dists$. The specific formula for `get_alpha` is not detailed in the source, but it typically involves a sigmoid or similar function mapping SDF to opacity, modulated by the angle between the ray and normal.
   - $\alpha = \text{get\_alpha}(S, N, D, dists)$.
6. **Compute Color**:
   - Compute RGB color using the feature vector $F$, ray direction $D$, and normal $N$: $RGB = \text{texture}(F, D, N)$.
7. **Compute Rendering Weights**:
   - Compute weights from alpha values. The weight $w_i$ for sample $i$ along a ray is typically calculated as $w_i = \alpha_i \prod_{j < i} (1 - \alpha_j)$, where $\alpha_j$ are the alphas of previous samples on the same ray.
8. **Accumulate Results**:
   - Accumulate opacity: $Opacity = \sum w_i$.
   - Accumulate depth: $Depth = \sum w_i \times midpoints_i$.
   - Accumulate RGB: $CompRGB = \sum w_i \times RGB_i$.
   - Accumulate normal: $CompNormal_{raw} = \sum w_i \times N_i$.
   - Normalize the composite normal: $CompNormal = \text{normalize}(CompNormal_{raw})$.
9. **Construct Output**:
   - Create a validity mask: $Valid = Opacity > 0$.
   - If training, store intermediate tensors for gradient computation.
10. **Handle Background**:
    - If `config.learned_background` is true, call the background rendering function `forward_bg_`.
    - Otherwise, create a background output with RGB set to `config.background_color` and zeroed sample counts/validity.
    - Combine foreground and background: $FinalRGB = CompRGB + BackgroundRGB \times (1.0 - Opacity)$.
    - Update `num_samples` and `rays_valid` by addition and logical OR, respectively.
11. **Return Output**: Return the combined output dictionary.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_pass(rays_o, rays_d, t_starts, t_ends, ray_indices, n_rays, config, geometry_net, texture_net, background_color):
    """
    Forward pass for Instant-NSR-PL rendering.
    """
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
    alpha = get_alpha(sdf, normal, t_dirs, dists)
    alpha = alpha[..., None]  # Shape: (..., 1)
    
    # 6. Compute Color
    rgb = texture_net(feature, t_dirs, normal)
    
    # 7. Compute Rendering Weights
    # Note: render_weight_from_alpha typically computes cumulative product of (1-alpha)
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
        'num_samples': weights.shape[0] // n_rays  # Approximate, exact count depends on implementation
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
            'num_samples': torch.zeros_like(opacity)
        }
        
    # Combine foreground and background
    final_rgb = out['comp_rgb'] + out_bg['comp_rgb'] * (1.0 - out['opacity'])
    out['comp_rgb'] = final_rgb
    out['num_samples'] = out['num_samples'] + out_bg['num_samples']
    out['rays_valid'] = out['rays_valid'] | out_bg['rays_valid']
    
    return out

def get_alpha(sdf, normal, ray_dir, dists):
    """
    Compute opacity from SDF, normal, ray direction, and segment distance.
    Implementation detail: Typically involves a sigmoid of SDF scaled by a factor,
    possibly modulated by the angle between normal and ray direction.
    """
    # Placeholder for specific formula not fully detailed in source
    # Common approach: alpha = sigmoid(sdf * sigma) * (1 - exp(-sigma * dist))
    # Or: alpha = sigmoid(sdf) * (1 - exp(-dists))
    # The source calls self.get_alpha, so we assume it handles the specific Instant-NSR logic.
    # For implementation, one might use:
    sigma = 10.0  # Example parameter, not given in source
    alpha = torch.sigmoid(sdf * sigma) * (1.0 - torch.exp(-sigma * dists))
    return alpha

def render_weight_from_alpha(alpha, ray_indices, n_rays):
    """
    Compute rendering weights from alpha values.
    """
    # Compute cumulative product of (1 - alpha) along rays
    # weights[i] = alpha[i] * prod_{j < i, same ray} (1 - alpha[j])
    # This requires sorting samples by ray and then computing cumulative product.
    # Simplified logic:
    # 1. Sort samples by ray_indices
    # 2. Compute cumulative product of (1-alpha)
    # 3. Multiply by alpha
    # 4. Restore original order
    pass # Implementation details for cumulative product along rays are complex and not fully specified in source

def accumulate_along_rays(weights, ray_indices, values, n_rays):
    """
    Accumulate values along rays using weights.
    """
    # Initialize accumulators
    accum = torch.zeros(n_rays, *values.shape[1:], device=values.device)
    # Scatter add
    accum.scatter_add_(0, ray_indices.unsqueeze(-1).expand_as(values), weights.unsqueeze(-1) * values)
    return accum
```

### Parameters
- `config.geometry.grad_type`: String, 'finite_difference' or 'analytical'. Determines how SDF gradients are computed.
- `config.learned_background`: Boolean, whether to use a learned background model.
- `config.background_color`: Tensor, color used for background if not learned.
- `config.ray_chunk`: Integer, batch size for ray processing during evaluation to manage memory.

### Edge Cases and Pitfalls
- **Precision**: Finite difference gradient computation can be sensitive to step size. If `grad_type` is 'finite_difference', the step size must be chosen carefully to balance accuracy and numerical stability.
- **Boundaries**: Ray origins and directions must be correctly aligned with the coordinate system used by the geometry network.
- **Degenerate Input**: Rays with `t_starts == t_ends` should be handled to avoid division by zero or invalid segment distances.
- **Background Handling**: If `learned_background` is false, the background color must be consistent with the training data to avoid artifacts.

## Subtopics

### Differentiable Silhouette Rendering
Renders binary silhouettes using SDFs. Choose when only shape boundary is needed, not color or normals. [Link](4322-differentiable-silhouette-rendering.md)

### SDF Direct Diffuse Integrator
Integrates diffuse shading directly from SDFs. Choose for physically based diffuse rendering without learned texture networks. [Link](4467-sdf-direct-diffuse-integrator.md)

## Operating characteristics
- **Time Complexity**: Linear with respect to the number of samples per ray and the number of rays. $O(N \times S)$, where $N$ is the number of rays and $S$ is the number of samples per ray.
- **Memory Complexity**: Linear with respect to the number of samples. $O(N \times S)$.
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales with the number of rays and samples. Chunking is used during evaluation to manage memory.
- **Strengths**: Differentiable, supports gradient computation with respect to scene parameters, handles complex geometry via SDFs.
- **Limitations**: Requires careful tuning of SDF query parameters, finite difference gradients can be noisy.
- **When to Use**: When differentiable rendering of complex 3D scenes with learned textures is required.
- **When Not to Use**: When real-time performance is critical and differentiability is not needed, or when analytical gradients are preferred over finite differences.

## Background
The Instant-NSR-PL Forward Pass is part of the Instant-Neural SDF Rendering (Instant-NSR) method, introduced by Guo et al. in the repository `bennyguo/instant-nsr-pl` [S1, unofficial]. It builds on the principles of Neural SDFs and differentiable rendering, aiming to provide a fast and flexible framework for rendering 3D scenes from 2D images. The method supports both analytical and finite-difference gradients for the SDF, allowing for flexibility in training and inference.

## Sources
[S1] bennyguo/instant-nsr-pl, https://github.com/bennyguo/instant-nsr-pl, fetched 2026-10-27 (unofficial)
