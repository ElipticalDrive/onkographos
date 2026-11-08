# DVGO Volume Rendering Forward

Also called: DVGO Forward Pass, Direct Voxel Grid Opacity Forward

## Overview
Performs volume rendering for a batch of rays, including sampling, density/alpha computation, transmittance calculation, color estimation, and ray marching [S1, unofficial]. It sets itself apart by using a direct voxel grid opacity representation where density is queried from a network and converted to alpha, often utilizing mask caching for free-space pruning and fast color thresholds for early termination [S1, unofficial].

## How it works
**Inputs:**
- `ray_pts`: Tensor of shape `(N, 3)` containing 3D coordinates of sample points on rays.
- `ray_id`: Tensor of shape `(N,)` mapping each point to its parent ray index.
- `step_id`: Tensor of shape `(N,)` or depth values associated with points.
- `viewdirs`: Tensor of shape `(N, 3)` viewing directions for each point.
- `bg`: Background color tensor.
- `render_kwargs`: Configuration dictionary containing `stepsize` and other parameters.
- `mask_cache`: Optional boolean mask indicating free space.
- `N`: Total number of rays.

**Outputs:**
- `rgb`: Rendered RGB image tensor.
- `depth`: (Optional) Rendered depth tensor.

**Data Structures:**
- `ray_pts`: Float tensor `(N, 3)`.
- `ray_id`: Long/Int tensor `(N,)`.
- `step_id`: Float tensor `(N,)`.
- `viewdirs`: Float tensor `(N, 3)`.

**Parameters:**
- `interval`: Calculated as `render_kwargs['stepsize'] * self.voxel_size_ratio` [S1, unofficial].
- `fast_color_thres`: Threshold for early termination. If `> 0` and `use_fourier_grid` is false, points with `alpha > fast_color_thres` or `weights > fast_color_thres` are filtered [S1, unofficial].
- `use_fourier_grid`: Boolean flag. If true, mask caching and fast color thresholding are skipped [S1, unofficial].
- `rgbnet_direct`: Boolean. If true, `k0` is used directly as view feature; otherwise `k0` is split into diffuse (first 3 channels) and view (remaining channels) [S1, unofficial].
- `rgbnet_full_implicit`: Boolean. If false, color feature `k0` is queried from a separate network [S1, unofficial].
- `render_depth`: Boolean. If true, depth is computed [S1, unofficial].

**Steps:**

1. **Sampling:** Sample points on rays using `self.sample_ray` [S1, unofficial].
2. **Free Space Pruning:** If `mask_cache` exists and `use_fourier_grid` is false, filter `ray_pts`, `ray_id`, and `step_id` by the mask to skip known free space [S1, unofficial].
3. **Density Query:** Query density from the density network [S1, unofficial].
4. **Alpha Activation:** Activate density to alpha using `self.activate_density` [S1, unofficial].
5. **Fast Color Threshold (Alpha):** If `fast_color_thres > 0` and `use_fourier_grid` is false, filter points where `alpha > fast_color_thres` [S1, unofficial].
6. **Transmittance Weights:** Compute accumulated transmittance weights using `Alphas2Weights.apply` with `alpha`, `ray_id`, and `N` [S1, unofficial].
7. **Fast Color Threshold (Weight):** If `fast_color_thres > 0` and `use_fourier_grid` is false, filter points where `weights > fast_color_thres` [S1, unofficial].
8. **Color Feature Query:** Query color feature `k0` from the `k0` network if `rgbnet_full_implicit` is false [S1, unofficial].
9. **View-Dependent Color:**
   - If `rgbnet` exists, compute view-dependent color.
   - If `rgbnet_direct` is true, use `k0` as view feature.
   - Otherwise, split `k0` into diffuse (`k0_diffuse`, first 3 channels) and view (`k0_view`, remaining channels) [S1, unofficial].
10. **View Direction Embedding:**
    - Flatten `viewdirs`.
    - Multiply by `viewfreq`.
    - Concatenate sin/cos embeddings.
    - Flatten and index by `ray_id` [S1, unofficial].
11. **RGB Logit Computation:** Concatenate `k0_view` and `viewdirs_emb`, pass through `rgbnet` to get `rgb_logit` [S1, unofficial].
12. **Final RGB:**
    - If `rgbnet_direct`, `rgb = sigmoid(rgb_logit)`.
    - Else, `rgb = sigmoid(rgb_logit + k0_diffuse)` [S1, unofficial].
13. **Marched RGB:** Compute marched RGB using `segment_coo`: sum of `(weights * rgb)` grouped by `ray_id`, plus background contribution (`alphainv_last * bg`) [S1, unofficial].
14. **Depth (Optional):** If `render_depth` is true, compute depth using `segment_coo`: sum of `(weights * step_id)` grouped by `ray_id` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dvgo_forward(ray_pts, ray_id, step_id, viewdirs, bg, render_kwargs, 
                 mask_cache=None, N=0, fast_color_thres=0.0, 
                 use_fourier_grid=False, rgbnet_direct=False, 
                 rgbnet_full_implicit=False, render_depth=False):
    """
    Performs DVGO volume rendering forward pass.
    """
    # 1. Sampling (Assumed pre-computed or handled by caller)
    # ray_pts, ray_id, step_id are inputs
    
    # 2. Free Space Pruning
    if mask_cache is not None and not use_fourier_grid:
        # Filter points based on mask_cache
        mask = mask_cache[ray_id] # Assuming mask_cache is indexed by ray_id or point index
        # Note: Exact masking logic depends on mask_cache layout. 
        # Typically, mask_cache indicates if a voxel/ray segment is free.
        # For this implementation, we assume a boolean mask for valid points.
        valid = mask_cache[ray_pts] # Placeholder for actual mask lookup
        ray_pts = ray_pts[valid]
        ray_id = ray_id[valid]
        step_id = step_id[valid]
        viewdirs = viewdirs[valid]

    # 3. Density Query
    density = query_density_network(ray_pts) # Defined elsewhere

    # 4. Alpha Activation
    alpha = activate_density(density) # Defined elsewhere, e.g., exp(-density * interval)

    # 5. Fast Color Threshold (Alpha)
    if fast_color_thres > 0 and not use_fourier_grid:
        valid = alpha > fast_color_thres
        ray_pts = ray_pts[valid]
        ray_id = ray_id[valid]
        step_id = step_id[valid]
        viewdirs = viewdirs[valid]
        alpha = alpha[valid]

    # 6. Transmittance Weights
    weights = alphas2weights(alpha, ray_id, N) # Segment-wise accumulation

    # 7. Fast Color Threshold (Weight)
    if fast_color_thres > 0 and not use_fourier_grid:
        # Filter points where weights > fast_color_thres
        # Note: This step is ambiguous in sources. Usually, early termination 
        # stops marching, but here it filters points.
        valid = weights > fast_color_thres
        ray_pts = ray_pts[valid]
        ray_id = ray_id[valid]
        step_id = step_id[valid]
        viewdirs = viewdirs[valid]
        alpha = alpha[valid]
        weights = weights[valid]

    # 8. Color Feature Query
    if not rgbnet_full_implicit:
        k0 = query_k0_network(ray_pts) # Defined elsewhere
    else:
        k0 = None # Or handled differently

    # 9. View-Dependent Color Setup
    if rgbnet_direct:
        k0_view = k0
        k0_diffuse = None
    else:
        k0_diffuse = k0[:, :3]
        k0_view = k0[:, 3:]

    # 10. View Direction Embedding
    viewdirs_flat = viewdirs.view(-1, 3)
    viewdirs_emb = positional_embedding(viewdirs_flat, viewfreq) # Defined elsewhere
    # Index by ray_id if necessary, but viewdirs_emb is per-point
    # The source says "index by ray_id", which might imply gathering if viewdirs_emb is precomputed per ray
    # Assuming viewdirs_emb is computed per point for simplicity
    viewdirs_emb = viewdirs_emb # Shape (N, emb_dim)

    # 11. RGB Logit Computation
    if rgbnet_direct:
        rgb_logit = rgbnet(k0_view, viewdirs_emb)
    else:
        rgb_logit = rgbnet(k0_view, viewdirs_emb)

    # 12. Final RGB
    if rgbnet_direct:
        rgb = sigmoid(rgb_logit)
    else:
        rgb = sigmoid(rgb_logit + k0_diffuse)

    # 13. Marched RGB
    # segment_coo sums weights * rgb grouped by ray_id
    rgb_marched = segment_coo(weights * rgb, ray_id, N)
    
    # Background contribution
    # alphainv_last is the accumulated transmittance at the end of each ray
    alphainv_last = compute_alphainv_last(alpha, ray_id, N) # Defined elsewhere
    rgb_final = rgb_marched + alphainv_last * bg

    # 14. Depth (Optional)
    depth = None
    if render_depth:
        depth = segment_coo(weights * step_id, ray_id, N)

    return rgb_final, depth
```

**Edge Cases and Pitfalls:**
- **Precision:** Floating point precision in transmittance accumulation can lead to artifacts.
- **Boundaries:** Mask caching must correctly handle boundary voxels.
- **Degenerate Input:** Empty rays or rays with no valid points should be handled gracefully.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of sample points $N$.
- **Memory Complexity:** Linear with respect to $N$ for storing intermediate tensors.
- **Performance:** Fast due to early termination and mask caching.
- **Strengths:** Real-time rendering capability, efficient memory usage.
- **Limitations:** Dependent on voxel grid resolution and network capacity.

## Background
DVGO (Direct Voxel Grid Opacity) is a method for real-time volume rendering. It was introduced by [S1, unofficial] as part of the UnboundedNeRFPytorch repository. It builds on the idea of direct voxel grid opacity representation, allowing for efficient rendering without the need for complex ray marching structures.

## Sources
[S1] sjtuytc/UnboundedNeRFPytorch, https://github.com/sjtuytc/UnboundedNeRFPytorch, fetched 2026-10-26 (unofficial)
