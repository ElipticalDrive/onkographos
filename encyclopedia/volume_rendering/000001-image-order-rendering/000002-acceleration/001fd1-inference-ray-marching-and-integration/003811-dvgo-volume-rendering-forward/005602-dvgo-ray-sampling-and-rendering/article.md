# DVGO Ray Sampling and Rendering

Also called: DVGO forward pass, Direct Voxel Grid Optimization rendering

## Overview
This method performs the forward pass for Direct Voxel Grid Optimization (DVGO), computing the final RGB color, depth, and optional segmentation masks for a batch of rays by marching through a dense voxel grid. It differs from NeRF Volume Rendering by using a pre-allocated 3D voxel grid for density and color storage, enabling faster training and inference via direct memory access, while maintaining the volume rendering integral formulation. It builds on [DVGO Volume Rendering Forward](8157-dvgo-volume-rendering-forward.md).

## How it works
**Inputs:**
- `rays_o`: Ray origins, shape $[N, 3]$.
- `rays_d`: Ray directions, shape $[N, 3]$.
- `viewdirs`: Normalized view directions, shape $[N, 3]$.
- `render_kwargs`: Dictionary containing `near`, `far`, `stepsize`, `bg` (background color), `inverse_y`, `flip_x`, `flip_y`, `ndc`.
- `model`: The DVGO model instance containing:
  - `xyz_min`, `xyz_max`: Bounding box of the voxel grid.
  - `voxel_size`: Physical size of one voxel.
  - `voxel_size_ratio`: Ratio used for step calculation.
  - `density`: Voxel density grid.
  - `k0`: Voxel color/density feature grid (if `rgbnet_full_implicit` is false).
  - `rgbnet`: MLP for view-dependent color (optional).
  - `rgbnet_direct`: Boolean flag for direct color output.
  - `seg_mask_grid`: Segmentation mask grid (optional).
  - `dual_seg_mask_grid`: Dual segmentation mask grid (optional, used in 'fine' mode).
  - `mask_cache`: Binary cache for valid voxels (optional).
  - `num_objects`: Number of segmentation classes.
  - `mode`: 'fine' or other.
  - `fast_color_thres`: Minimum threshold for color filtering.
  - `render_fct`: Threshold for alpha/weight filtering (default 0).

**Outputs:**
- `rgb_marched`: Rendered RGB colors, shape $[N, 3]$.
- `depth`: Rendered depth (if requested).
- `alphainv_last`: Transmittance at the end of the ray, shape $[N]$.
- `weights`: Rendering weights for sampled points (filtered).
- `seg_mask_marched`: Segmentation masks, shape $[N, \text{num\_objects}]$.
- `dual_seg_mask_marched`: Dual segmentation masks (if 'fine' mode).
- `raw_alpha`, `raw_rgb`, `ray_id`, `step_id`: Intermediate filtered data.

**Data Structures:**
- **Voxel Grids**: Dense 3D tensors. `density` stores raw density values. `k0` stores raw color features (RGB + optional diffuse/specular components). `seg_mask_grid` stores per-object probabilities.
- **Ray Data**: Batched tensors for origins, directions, and view vectors.

**Steps:**

1. **Ray-Bounding Box Intersection:**
   Compute the intersection interval $[t_{min}, t_{max}]$ of each ray with the voxel grid bounding box $[xyz_{min}, xyz_{max}]$.
   Let $vec = rays\_d$.
   $$ rate\_a = \frac{xyz_{min} - rays\_o}{vec} $$
   $$ rate\_b = \frac{xyz_{max} - rays\_o}{vec} $$
   $$ t_{min} = \text{clamp}(\max(\min(rate\_a, rate\_b), \text{axis}=2), \text{min}=near, \text{max}=far) $$
   $$ t_{max} = \text{clamp}(\min(\max(rate\_a, rate\_b), \text{axis}=2), \text{min}=near, \text{max}=far) $$
   If $t_{min} \ge t_{max}$, the ray does not intersect the box or is outside the near/far planes.

2. **Point Sampling:**
   Calculate the step distance for sampling.
   $$ step = stepsize \times voxel\_size \times rng $$
   where $rng$ is a random variable uniform in $[0, 1]$ for jittering.
   Compute interpolation positions along the ray:
   $$ interpx = t_{min} + \frac{step}{\|rays\_d\|} $$
   Compute ray points:
   $$ rays\_pts = rays\_o + rays\_d \times interpx $$

3. **Filtering by Bounding Box and Cache:**
   Use a CUDA kernel `sample_pts_on_rays` to:
   - Determine which points lie within $[xyz_{min}, xyz_{max}]$ and $[near, far]$.
   - Generate `mask_outbbox` (points outside).
   - Assign `ray_id` and `step_id` to each valid point.
   - Return `N_steps` (number of valid steps per ray).
   Filter to `mask_inbbox = ~mask_outbbox`.
   If `mask_cache` exists, further filter points using `mask_cache(rays_pts[mask_inbbox])`.

4. **Alpha Thresholding (Optimization):**
   If `render_fct > 0`:
   Compute alpha for all points.
   Create mask `mask = (alpha > render_fct)`.
   Filter `ray_pts`, `ray_id`, `step_id`, `density`, `alpha` by this mask.
   Note: `render_fct` is set to $\max(render\_fct, fast\_color\_thres)$.

5. **Alpha to Weight Conversion:**
   Compute rendering weights from alpha values using the `Alphas2Weights` function, which implements the standard volume rendering accumulation:
   $$ \alpha_i = 1 - \exp(-\sigma_i \Delta_i) $$
   $$ w_i = \alpha_i \prod_{j=0}^{i-1} (1 - \alpha_j) $$
   This is implemented efficiently using `Alphas2Weights.apply(alpha, ray_id, N)`.
   If `render_fct > 0`, filter `weights`, `alpha`, `ray_pts`, `ray_id`, `step_id` by `mask = (weights > render_fct)`.

6. **Segmentation Mask Query:**
   If `seg_mask_grid.grid.requires_grad`:
   Enable gradient tracking.
   Query `mask_pred = seg_mask_grid(rays_pts)`.
   If `mode == 'fine'`, also query `dual_mask_pred = dual_seg_mask_grid(rays_pts)`.
   If not requiring grad, query directly.

7. **Color Query:**
   Query raw color features `k0` from the voxel grid: `k0 = k0(rays_pts)`.
   If `rgbnet` is None:
   $$ rgb = \text{sigmoid}(k0) $$
   If `rgbnet` exists:
   - Determine view-dependent input:
     If `rgbnet_direct`: `k0_view = k0`.
     Else: `k0_view = k0[:, 3:]`, `k0_diffuse = k0[:, :3]`.
   - Compute view direction embedding:
     $$ viewdirs\_emb = \text{concat}([viewdirs, \sin(viewdirs \times \pi \times 2^{\text{freq}}), \cos(viewdirs \times \pi \times 2^{\text{freq}})]) $$
     (Frequency multiplication is applied as per standard positional encoding).
   - Flatten and index `viewdirs_emb` by `ray_id`.
   - Concatenate features: `rgb_feat = concat([k0_view, viewdirs_emb], dim=-1)`.
   - Compute logit: `rgb_logit = rgbnet(rgb_feat)`.
   - Compute final RGB:
     If `rgbnet_direct`: `rgb = sigmoid(rgb_logit)`.
     Else: `rgb = sigmoid(rgb_logit + k0_diffuse)`.

8. **RGB Accumulation:**
   Accumulate RGB using scatter sum:
   $$ rgb\_marched = \text{segment\_coo}(src = weights \times rgb, index = ray\_id, out = \text{zeros}([N, 3]), reduce = \text{'sum'}) $$

9. **Segmentation Accumulation:**
   If `num_objects == 1`:
   $$ seg\_mask\_marched = \text{segment\_coo}(src = weights \times mask\_pred, index = ray\_id, out = \text{zeros}([N, 1]), reduce = \text{'sum'}) $$
   If `mode == 'fine'`, also compute `dual_seg_mask_marched`.
   If `num_objects != 1`:
   $$ seg\_mask\_marched = \text{segment\_coo}(src = weights \times mask\_pred, index = ray\_id, out = \text{zeros}([N, num\_objects]), reduce = \text{'sum'}) $$
   If `mode == 'fine'`, also compute `dual_seg_mask_marched`.

10. **Background Addition:**
    $$ rgb\_marched += alphainv\_last \times bg $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def dvgo_ray_sampling_and_rendering(rays_o, rays_d, viewdirs, render_kwargs, model):
    """
    Performs DVGO ray sampling and rendering.
    """
    device = rays_o.device
    N = rays_o.shape[0]
    near = render_kwargs['near']
    far = render_kwargs['far']
    stepsize = render_kwargs['stepsize']
    bg = render_kwargs['bg']
    
    xyz_min = model.xyz_min
    xyz_max = model.xyz_max
    voxel_size = model.voxel_size
    voxel_size_ratio = model.voxel_size_ratio
    render_fct = model.render_fct
    fast_color_thres = model.fast_color_thres
    
    # Step 1: Ray-Bounding Box Intersection
    vec = rays_d
    rate_a = (xyz_min - rays_o) / vec
    rate_b = (xyz_max - rays_o) / vec
    
    t_min = torch.minimum(rate_a, rate_b).amax(dim=-1)
    t_max = torch.maximum(rate_a, rate_b).amin(dim=-1)
    
    t_min = torch.clamp(t_min, min=near, max=far)
    t_max = torch.clamp(t_max, min=near, max=far)
    
    # Handle rays that don't intersect or are outside near/far
    valid_mask = t_min < t_max
    if not valid_mask.any():
        # Return zeros or background for all rays
        return {
            'rgb_marched': torch.zeros(N, 3, device=device),
            'alphainv_last': torch.ones(N, device=device),
            'depth': torch.zeros(N, device=device),
            'weights': torch.zeros(0, device=device),
            'raw_alpha': torch.zeros(0, device=device),
            'raw_rgb': torch.zeros(0, device=device),
            'ray_id': torch.zeros(0, dtype=torch.long, device=device),
            'seg_mask_marched': torch.zeros(N, model.num_objects, device=device),
            'dual_seg_mask_marched': torch.zeros(N, model.num_objects, device=device) if model.mode == 'fine' else None
        }

    # Step 2: Point Sampling
    # Generate random jitter for each step
    # Note: In practice, N_steps is determined by the CUDA kernel
    # Here we simulate the logic for clarity, assuming we have a way to get N_steps
    # The actual implementation uses a CUDA kernel to determine valid points
    
    # Step 3: Filter by Bounding Box and Cache
    # Call CUDA kernel: sample_pts_on_rays
    # Returns: ray_pts, mask_outbbox, ray_id, step_id, N_steps, t_min, t_max
    # For pseudocode, we assume these are computed
    # ray_pts, mask_outbbox, ray_id, step_id, N_steps, t_min, t_max = sample_pts_on_rays_cuda(...)
    
    # Since we can't call CUDA, we describe the logic:
    # 1. Compute step distance: step = stepsize * voxel_size * rng
    # 2. Compute interpx = t_min + step / ||rays_d||
    # 3. Compute rays_pts = rays_o + rays_d * interpx
    # 4. Filter points within xyz_min/xyz_max and near/far
    # 5. Apply mask_cache if exists
    
    # For the sake of the pseudocode, we assume we have filtered points
    # ray_pts_valid, ray_id_valid, step_id_valid, N_steps_valid = ...
    
    # Step 4: Alpha Thresholding
    # Compute alpha from density
    # density = model.density(ray_pts_valid)
    # shift = model.shift # Assuming shift is a parameter
    # interval = stepsize * voxel_size_ratio
    # alpha = 1 - (1 + torch.exp(density + shift))**(-interval)
    # This is equivalent to: alpha = 1 - torch.exp(-torch.nn.functional.softplus(density + shift) * interval)
    
    # If render_fct > 0, filter
    if render_fct > 0:
        render_fct = max(render_fct, fast_color_thres)
        # mask = (alpha > render_fct)
        # Filter ray_pts, ray_id, step_id, density, alpha by mask
    
    # Step 5: Alpha to Weight Conversion
    # weights, alphainv_last = Alphas2Weights.apply(alpha, ray_id, N)
    # If render_fct > 0, filter weights, alpha, ray_pts, ray_id, step_id by (weights > render_fct)
    
    # Step 6: Segmentation Mask Query
    seg_mask_marched = torch.zeros(N, model.num_objects, device=device)
    dual_seg_mask_marched = None
    
    if model.seg_mask_grid is not None:
        if model.seg_mask_grid.grid.requires_grad:
            model.seg_mask_grid.grid.requires_grad_(True)
            mask_pred = model.seg_mask_grid(ray_pts_valid)
            if model.mode == 'fine':
                dual_mask_pred = model.dual_seg_mask_grid(ray_pts_valid)
        else:
            mask_pred = model.seg_mask_grid(ray_pts_valid)
            if model.mode == 'fine':
                dual_mask_pred = model.dual_seg_mask_grid(ray_pts_valid)
        
        # Accumulate segmentation masks
        if model.num_objects == 1:
            seg_mask_marched = segment_coo(
                src=(weights.unsqueeze(-1) * mask_pred.unsqueeze(-1)),
                index=ray_id_valid,
                out=torch.zeros(N, 1, device=device),
                reduce='sum'
            )
            if model.mode == 'fine':
                dual_seg_mask_marched = segment_coo(
                    src=(weights.unsqueeze(-1) * dual_mask_pred.unsqueeze(-1)),
                    index=ray_id_valid,
                    out=torch.zeros(N, 1, device=device),
                    reduce='sum'
                )
        else:
            seg_mask_marched = segment_coo(
                src=(weights.unsqueeze(-1) * mask_pred),
                index=ray_id_valid,
                out=torch.zeros(N, model.num_objects, device=device),
                reduce='sum'
            )
            if model.mode == 'fine':
                dual_seg_mask_marched = segment_coo(
                    src=(weights.unsqueeze(-1) * dual_mask_pred),
                    index=ray_id_valid,
                    out=torch.zeros(N, model.num_objects, device=device),
                    reduce='sum'
                )

    # Step 7: Color Query
    k0 = model.k0(ray_pts_valid)
    
    if model.rgbnet is None:
        rgb = torch.sigmoid(k0)
    else:
        if model.rgbnet_direct:
            k0_view = k0
        else:
            k0_view = k0[:, 3:]
            k0_diffuse = k0[:, :3]
        
        # View direction embedding
        # Assuming freq_mult is a parameter or fixed
        freq_mult = 1 # Placeholder for actual frequency multiplication logic
        viewdirs_emb = torch.cat([
            viewdirs,
            torch.sin(viewdirs * freq_mult * torch.pi),
            torch.cos(viewdirs * freq_mult * torch.pi)
        ], dim=-1)
        
        # Flatten and index by ray_id
        # Note: viewdirs_emb needs to be indexed by ray_id_valid
        viewdirs_emb_indexed = viewdirs_emb[ray_id_valid]
        
        rgb_feat = torch.cat([k0_view, viewdirs_emb_indexed], dim=-1)
        rgb_logit = model.rgbnet(rgb_feat)
        
        if model.rgbnet_direct:
            rgb = torch.sigmoid(rgb_logit)
        else:
            rgb = torch.sigmoid(rgb_logit + k0_diffuse)

    # Step 8: RGB Accumulation
    rgb_marched = segment_coo(
        src=(weights.unsqueeze(-1) * rgb),
        index=ray_id_valid,
        out=torch.zeros(N, 3, device=device),
        reduce='sum'
    )

    # Step 10: Background Addition
    rgb_marched += (alphainv_last.unsqueeze(-1) * bg)

    return {
        'rgb_marched': rgb_marched,
        'depth': None, # Depth computation depends on specific implementation
        'alphainv_last': alphainv_last,
        'weights': weights,
        'raw_alpha': alpha,
        'raw_rgb': rgb,
        'ray_id': ray_id_valid,
        'seg_mask_marched': seg_mask_marched,
        'dual_seg_mask_marched': dual_seg_mask_marched
    }

def segment_coo(src, index, out, reduce='sum'):
    """
    PyTorch scatter operation equivalent to COO segment sum.
    """
    # This is a simplified representation. In practice, torch.scatter_add is used.
    # out.scatter_add_(dim=0, index=index.unsqueeze(-1).expand_as(src), src=src)
    # For multi-dimensional out, we need to handle each channel separately or use advanced indexing.
    # Here we assume a simple 1D index for rays.
    for i in range(out.shape[1]):
        out[:, i].scatter_add_(0, index, src[:, i])
    return out
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating point precision issues can arise when computing `rate_a` and `rate_b` for rays nearly parallel to the bounding box faces. Clamping to `near` and `far` helps.
- **Boundaries:** Rays that do not intersect the bounding box must be handled to avoid invalid memory access or NaN values.
- **Degenerate Input:** Empty batches or rays with zero length should be checked.
- **Memory:** Large batches can exhaust GPU memory. Splitting rays into chunks (e.g., 8192) is recommended.

## Operating characteristics
- **Time Complexity:** $O(N \cdot S)$, where $N$ is the number of rays and $S$ is the number of samples per ray. The dominant cost is the voxel grid lookups and the scatter operations.
- **Memory Complexity:** $O(V + N \cdot S)$, where $V$ is the number of voxels. The voxel grids are stored in memory, and intermediate tensors for ray points and features are allocated per batch.
- **Measured Performance:** DVGO is generally faster to train than NeRF due to the dense voxel grid allowing for efficient memory access and parallelization. Inference speed is comparable to NeRF but can be faster with optimized voxel traversal.
- **Strengths:** Fast training, efficient memory access, supports view-dependent effects.
- **Limitations:** Memory usage scales with voxel resolution, which can be high for large scenes. Less flexible than NeRF for representing complex geometry without high resolution.
- **When to Use:** When training speed is a priority and the scene can be represented well with a voxel grid. Suitable for real-time applications if optimized.
- **When Not to Use:** For scenes with very fine details that require extremely high voxel resolution, or for scenes with complex topology that is better represented by implicit functions.

## Background
DVGO was introduced in the paper "Direct Voxel Grid Optimization for Super-Resolution Compact 3D Gaussian Splatting" (Note: The source [S1, unofficial] refers to "SegmentAnythingin3D" which uses DVGO, but the original DVGO paper is "Direct Voxel Grid Optimization" by Yu et al., 2021. The notes provided are from the GitHub repository [S1, unofficial] which implements DVGO for segmentation). The method was developed to address the slow training times of NeRF by using a dense voxel grid instead of a neural network for density and color representation. It has been used in various 3D reconstruction and segmentation tasks.

## Sources
[S1] Jumpat/SegmentAnythingin3D, https://github.com/Jumpat/SegmentAnythingin3D, fetched 2026-10-26 (unofficial)
