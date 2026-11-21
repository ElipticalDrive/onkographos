# Background Rendering Backward Pass

Also called: Background Backpropagation, Background Rendering Kernel (Backward)

## Overview
The Background Rendering Backward Pass computes the gradients of the loss with respect to the background volumetric field (typically a sparse grid of spherical harmonic coefficients or density values) and the accumulated transmittance, given the rendered background color, ground truth, and ray parameters. It serves as the differentiable counterpart to the concentric sphere sampling and trilinear interpolation used in the forward background rendering pass. Unlike the foreground ray casting which traverses a dense or sparse voxel grid, this method specifically handles the unbounded background field, often using a separate sparse grid structure [S1, unofficial]. It is distinct from the general Ray Tracing Backward Pass by isolating the gradient computation for the background component, allowing for specialized handling of RGB normalization and sparsity regularization specific to the background layer [S2, unofficial].

## How it works
The method is implemented as a CUDA kernel that processes rays in parallel. It assumes the forward pass has already computed the accumulated transmittance (`log_transmit`) and the final background color contribution. The backward pass iterates through the same sampling steps as the forward pass to compute local gradients and accumulate them into the background data structure.

### Inputs and Outputs
**Inputs:**
*   `grid`: SparseGridSpec containing the background volumetric data (density/sigma and SH coefficients).
*   `rgb_gt`: Ground truth RGB tensor for the current batch.
*   `rgb_out`: Rendered background RGB tensor.
*   `rays`: RaysSpec containing ray origins, directions, and IDs.
*   `opt`: RenderOptions including `background_nlayers`, `step_size`, and `use_background`.
*   `log_transmit`: Tensor of accumulated log-transmittance values per ray.
*   `accum`: Tensor of accumulated color contributions per ray.
*   `sparsity_loss`: Boolean flag indicating if sparsity regularization is active.
*   `grad_output`: Gradient of the loss with respect to the final output color (RGB).

**Outputs:**
*   `grads.grad_background_out`: Gradient buffer for the background volumetric data.
*   `grads.grad_log_transmit`: (Optional) Gradient for accumulated transmittance.

### Data Structures
*   **SparseGridSpec**: Contains the layout for the background sparse grid. The exact internal layout (e.g., hash map keys, leaf node sizes) is defined by the parent `Background Volume Rendering with Sparse Grid` implementation [S1, unofficial].
*   **ConcentricSpheresIntersector**: Helper object for intersecting rays with concentric spheres. It stores `ray.origin` and `ray.dir`.
*   **SingleRaySpec**: Temporary structure holding interpolated values (`ray.pos`, `ray.l`, `invr_mid`) for the current sampling step.

### Parameters
*   `background_nlayers`: Number of layers in the background grid. Typical value: 32 [S2, unofficial].
*   `step_size`: Step size for concentric sphere sampling. Typical value: Not given in the sources.
*   `inner_radius`: Minimum radius for background sampling. Calculated as `fmaxf(_dist_ray_to_origin(ray.origin, ray.dir) + 1e-3f, 1.f)` [S3, unofficial].
*   `TRACE_RAY_BG_CUDA_THREADS`: Number of CUDA threads per block. Typical value: 256 [S1, unofficial].
*   `sparsity_loss`: Weight for Cauchy sparsity regularization. Passed as an argument [S3, unofficial].
*   `use_background`: Boolean flag to enable/disable the background backward pass [S1, unofficial].

### Steps
1.  **Initialization**:
    *   If `use_background` is false, return.
    *   Get `ray_id` for the current thread.
    *   **Early Exit**: If `log_transmit[ray_id] < -25.0f`, the ray is fully transparent or saturated; return early [S1, S2, unofficial].
    *   Initialize `total_color = 0.0f`.
    *   Initialize `ConcentricSpheresIntersector csi` with `ray.origin` and `ray.dir` [S3, unofficial].
    *   Calculate `inner_radius = fmaxf(_dist_ray_to_origin(ray.origin, ray.dir) + 1e-3f, 1.0f)` [S3, unofficial].
    *   Calculate `n_steps = int(grid.background_nlayers / opt.step_size) + 2` [S3, unofficial].
    *   Initialize `invr_last = 1.0f / inner_radius` [S3, unofficial].

2.  **Gradient Output Preparation**:
    *   Determine `grad_out` based on `grad_out_is_rgb` flag:
        *   If `grad_out_is_rgb` is true: `grad_out = resid * norm_factor`, where `norm_factor = 2.0f / (3.0f * Q)` and `Q` is the number of rays [S2, unofficial].
        *   Else: `grad_out = grad_output` [S2, unofficial].

3.  **Sampling Loop**:
    *   Loop `i` from `0` to `n_steps - 1`:
        *   Calculate radius `r = n_steps / (n_steps - i - 0.5)` [S3, unofficial].
        *   If `r < inner_radius` or `!csi.intersect(r, &t)`, continue to next iteration [S3, unofficial].
        *   Calculate position `ray.pos = origin + t * dir` [S3, unofficial].
        *   Calculate `invr_mid = _rnorm(ray.pos)` (inverse norm of position) [S3, unofficial].
        *   Normalize position: `ray.pos *= invr_mid` [S3, unofficial].
        *   Convert to equirectangular coordinates: `_unitvec2equirect(ray.pos)` [S3, unofficial].
        *   Calculate layer index and integer lattice `ray.l`, clamp to grid bounds, and compute fractional part `ray.pos` for interpolation [S3, unofficial].
        *   **Trilinear Interpolation of Sigma**:
            *   `sigma = trilerp_bg_one(grid, ray.l, ray.pos)` [S3, unofficial].
        *   **Compute Weight and Transmittance Update**:
            *   If `sigma > 0.0f`:
                *   `pcnt = ray.world_step * (invr_last - invr_mid) * sigma` [S3, unofficial].
                *   `weight = exp(log_transmit[ray_id]) * (1.0f - exp(-pcnt))` [S3, unofficial].
                *   `log_transmit[ray_id] -= pcnt` [S3, unofficial].
        *   **Color Accumulation**:
            *   For each RGB channel `i`:
                *   Interpolate color: `color = trilerp_bg_one(grid, ray.l, ray.pos) * C0 + 0.5f` (where `C0` is the base color coefficient) [S3, unofficial].
                *   `total_color += fmaxf(color, 0.0f) * grad_out[i]` [S3, unofficial].
        *   **Gradient Propagation to Background Data**:
            *   If `color > 0.0f`:
                *   Compute `curr_grad_color = C0 * weight * grad_out[i]` [S3, unofficial].
                *   Call `trilerp_backward_bg_one` to propagate `curr_grad_color` to `grads.grad_background_out` [S3, unofficial].

4.  **Accumulation and Sparsity Gradient**:
    *   **Accumulate Color Gradient**:
        *   `accum[ray_id] -= weight * total_color` [S1, unofficial].
    *   **Sparsity Loss Gradient** (if `sparsity_loss` is true):
        *   For each sampled `sigma`:
            *   `curr_grad_sigma = sparsity_loss * (4.0f * sigma / (1.0f + 2.0f * (sigma * sigma)))` [S1, unofficial].
            *   Call `trilerp_backward_bg_one` with `curr_grad_sigma` to update `grads.grad_background_out` [S1, unofficial].
    *   **Sigma Gradient from Color**:
        *   Compute `curr_grad_sigma = ray.world_step * (invr_last - invr_mid) * (total_color * exp(log_transmit[ray_id]) - accum[ray_id])` [S1, unofficial].
        *   Call `trilerp_backward_bg_one` with `curr_grad_sigma` to update `grads.grad_background_out` [S1, unofficial].

5.  **Kernel Launch**:
    *   Calculate blocks: `blocks = CUDA_N_BLOCKS_NEEDED(Q, TRACE_RAY_BG_CUDA_THREADS)` [S1, unofficial].
    *   Launch kernel with `blocks` and `TRACE_RAY_BG_CUDA_THREADS`.
    *   Check for CUDA errors using `CUDA_CHECK_ERRORS` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_background_backward(grid, rgb_gt, rgb_out, rays, opt, log_transmit, accum, sparsity_loss, grad_output):
    """
    Computes gradients for the background rendering pass.
    """
    if not opt.use_background:
        return

    Q = rays.num_rays
    threads = 256  # TRACE_RAY_BG_CUDA_THREADS
    blocks = (Q + threads - 1) // threads

    # Initialize gradient buffer
    grads = GradientBuffer(grid)

    # Launch CUDA kernel
    # Note: This is a conceptual representation of the CUDA kernel logic
    for ray_id in range(Q):
        ray = rays[ray_id]
        
        # Early exit check
        if log_transmit[ray_id] < -25.0:
            continue

        total_color = 0.0
        inner_radius = max(dist_ray_to_origin(ray.origin, ray.dir) + 1e-3, 1.0)
        n_steps = int(grid.background_nlayers / opt.step_size) + 2
        invr_last = 1.0 / inner_radius
        
        csi = ConcentricSpheresIntersector(ray.origin, ray.dir)
        
        # Prepare gradient output
        if grad_out_is_rgb:
            norm_factor = 2.0 / (3.0 * Q)
            resid = rgb_gt[ray_id] - rgb_out[ray_id]
            grad_out = resid * norm_factor
        else:
            grad_out = grad_output[ray_id]

        for i in range(n_steps):
            r = n_steps / (n_steps - i - 0.5)
            
            if r < inner_radius:
                continue
            
            t = csi.intersect(r)
            if t is None:
                continue
            
            ray_pos = ray.origin + t * ray.dir
            invr_mid = 1.0 / norm(ray_pos)
            ray_pos_normalized = ray_pos * invr_mid
            
            # Convert to equirectangular coordinates
            eq_coords = unitvec2equirect(ray_pos_normalized)
            
            # Get grid indices and fractional part
            lattice, frac = get_lattice_and_frac(eq_coords, grid.background_reso)
            
            # Trilinear interpolation of sigma
            sigma = trilerp_bg_one(grid, lattice, frac)
            
            if sigma > 0.0:
                pcnt = ray.world_step * (invr_last - invr_mid) * sigma
                weight = exp(log_transmit[ray_id]) * (1.0 - exp(-pcnt))
                log_transmit[ray_id] -= pcnt
                
                # Interpolate color
                color = trilerp_bg_one(grid, lattice, frac) * C0 + 0.5
                
                # Accumulate color gradient
                for ch in range(3):
                    total_color += max(color[ch], 0.0) * grad_out[ch]
                
                # Backpropagate to background data
                if color[ch] > 0.0:
                    curr_grad_color = C0 * weight * grad_out[ch]
                    trilerp_backward_bg_one(grads, lattice, frac, curr_grad_color)
                
                # Sparsity loss gradient
                if sparsity_loss:
                    curr_grad_sigma = sparsity_loss * (4.0 * sigma / (1.0 + 2.0 * sigma * sigma))
                    trilerp_backward_bg_one(grads, lattice, frac, curr_grad_sigma)
                
                # Sigma gradient from color
                curr_grad_sigma = ray.world_step * (invr_last - invr_mid) * (total_color * exp(log_transmit[ray_id]) - accum[ray_id])
                trilerp_backward_bg_one(grads, lattice, frac, curr_grad_sigma)
            
            invr_last = invr_mid

    return grads
```

### Edge Cases and Pitfalls
*   **Transmittance Saturation**: If `log_transmit[ray_id] < -25.0f`, the thread returns early to avoid numerical instability and wasted computation [S1, S2, unofficial].
*   **Precision**: The use of `float` (single precision) is standard in the CUDA implementation [S1, unofficial]. Care must be taken with the `exp` and `log` operations to avoid underflow/overflow.
*   **Boundary Conditions**: The `inner_radius` calculation ensures that rays do not sample inside the origin sphere, preventing division by zero or invalid intersections [S3, unofficial].
*   **CUDA Errors**: The kernel launch must be followed by error checking to catch invalid memory accesses or configuration errors [S1, unofficial].

## Subtopics

### [Background Volume Rendering with Sparse Grid](15366-background-volume-rendering-with-sparse-grid.md)
A technique for rendering unbounded or distant background elements using a separate sparse grid structure. It allows for efficient rendering by treating the background as a distinct volumetric field, enabling different resolution and sampling strategies compared to the foreground. Choose this when the scene has large unbounded backgrounds that do not fit efficiently into a dense voxel grid.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of rays $Q$ and the number of sampling steps $n_{steps}$. The complexity is $O(Q \cdot n_{steps})$.
*   **Memory Complexity**: Linear with respect to the number of rays for storing `log_transmit` and `accum` gradients, and proportional to the size of the background sparse grid for storing gradients.
*   **Performance**: The CUDA kernel is optimized for parallel execution, with 256 threads per block [S1, unofficial]. Performance scales with the number of rays and the complexity of the background grid traversal.
*   **Strengths**: Efficiently handles unbounded backgrounds; separates background and foreground computations for modular optimization; supports sparsity regularization.
*   **Limitations**: Requires a separate sparse grid structure for the background; performance may degrade if the background grid is very large or has high resolution.
*   **When to Use**: When rendering scenes with large, unbounded backgrounds (e.g., sky, distant geometry) that are distinct from the foreground scene.
*   **When Not to Use**: When the background is simple (e.g., a solid color or environment map) and does not require volumetric rendering.

## Background
The Background Rendering Backward Pass is part of the NeRF-Factory project by Kakaobrain, which includes custom CUDA kernels for sparse voxel volume rendering [S1, unofficial]. It is also implemented in Svox2 by sxyu, which uses spherical harmonics for background rendering [S2, unofficial]. The method builds on the principles of differentiable volume rendering, specifically adapting the backward pass for the concentric sphere sampling technique used for background rendering. The implementation in NeRF-Factory is described in the context of a PyTorch sparse voxel volume extension [S1, unofficial].

## Sources
[S1] kakaobrain/nerf-factory, https://github.com/kakaobrain/nerf-factory, fetched 2026-10-26 (unofficial)
[S2] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
[S3] Jittor/JNeRF, https://github.com/Jittor/JNeRF, fetched 2026-10-27 (unofficial)
