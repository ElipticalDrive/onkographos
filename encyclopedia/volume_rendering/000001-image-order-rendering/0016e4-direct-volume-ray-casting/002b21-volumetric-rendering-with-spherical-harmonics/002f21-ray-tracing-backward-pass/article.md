# Ray Tracing Backward Pass


## Overview
The Ray Tracing Backward Pass computes the gradients for spherical harmonic (SH) coefficients, ray origins, and ray directions required for optimizing Volumetric Rendering with Spherical Harmonics [S1, unofficial]. It serves as the differentiable counterpart to the forward ray casting kernel, enabling end-to-end training of 3D Gaussian or voxel-based scene representations [S1, unofficial]. It is distinguished from its sibling [Spherical Harmonic Volume Shading](5946-spherical-harmonic-volume-shading.md) by operating on the ray-tracing level to accumulate gradients across the entire ray path, including background contributions and regularization losses, rather than shading individual primitives [S1, unofficial].

## How it works
The method is implemented as a CUDA kernel (`render_ray_backward_kernel`) that processes rays in parallel. It computes gradients for the SH basis coefficients by backpropagating through the volume rendering integral and the SH evaluation functions.

**Inputs:**
*   `grad_output`: Gradient of the loss with respect to the rendered image (RGB or SH domain).
*   `color_cache`: The forward pass output (rendered colors) for each ray.
*   `ray_origins`, `ray_dirs`: Ray parameters.
*   `grid`: The 3D grid structure containing SH coefficients and density.
*   `opt`: Optimization options (e.g., `beta_loss`, `sparsity_loss` weights).
*   `log_transmit`: Log-transmittance values from the forward pass.
*   `accum`: Accumulated background color from the forward pass (if applicable).

**Outputs:**
*   `grads`: Gradients for SH coefficients, density, and ray parameters.
*   `grad_sphfunc_val`: Intermediate gradients for SH basis values.

**Data Structures:**
*   `sphfunc_val`: Shared memory array storing SH basis function values for the current ray step.
*   `grad_sphfunc_val`: Shared memory array storing gradients of SH basis values.
*   `ray_spec`: Structure containing ray origin and direction.
*   `temp_storage`: Warp-level reduction buffer.

**Parameters:**
*   `beta_loss`: Float weight for beta distribution sparsity loss [S1, unofficial].
*   `sparsity_loss`: Float weight for sparsity loss [S1, unofficial].
*   `Q`: Number of rays per block or warp dimension factor (used in normalization) [S1, unofficial].
*   `grid.sh_data_dim`: Dimension of the SH data grid [S1, unofficial].
*   `grid.basis_dim`: Dimension of the SH basis [S1, unofficial].

**Steps:**
1.  **Thread Identification:** Calculate global thread ID. Derive `ray_id`, `ray_blk_id`, and `lane_id`. If `lane_id >= grid.sh_data_dim`, return early [S1, unofficial].
2.  **Shared Memory Initialization:** Initialize `sphfunc_val`, `grad_sphfunc_val`, `ray_spec`, and `temp_storage` [S1, unofficial].
3.  **Ray Setup:** Set `ray_spec` from `ray_origins` and `ray_dirs`. Extract direction vector `vdir` [S1, unofficial].
4.  **Gradient Buffer Init:** If `lane_id < grid.basis_dim`, initialize `grad_sphfunc_val` to `0.0f` [S1, unofficial].
5.  **SH Evaluation:** Calculate spherical harmonic function values using `calc_sphfunc` for the current ray position [S1, unofficial].
6.  **Ray Bounds:** If `lane_id == 0`, find ray bounds using `ray_find_bounds` [S1, unofficial].
7.  **Gradient Normalization:** Compute `grad_out`:
    *   If `grad_out_is_rgb`: Calculate `resid = color_cache - grad_output`. Then `grad_out[i] = resid * norm_factor`, where `norm_factor = 2.0f / (3.0f * Q)` [S1, unofficial].
    *   Else: `grad_out[i] = grad_output[i]` [S1, unofficial].
8.  **Warp Synchronization:** Synchronize warps within the SH data dimension mask [S1, unofficial].
9.  **Volume Backward Trace:** Call `trace_ray_cuvol_backward` with `grad_out`, `color_cache`, `ray_spec`, `opt`, `lane_id`, `sphfunc_val`, `grad_sphfunc_val`, `temp_storage`, `log_transmit_in`, `beta_loss`, `sparsity_loss`, `grads`, and optional `accum`/`log_transmit` outputs [S1, unofficial]. This step accumulates gradients for density and SH coefficients along the ray.
10. **SH Backward Pass:** Call `calc_sphfunc_backward` to compute gradients for the spherical harmonic basis coefficients based on `grad_sphfunc_val` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for render_ray_backward_kernel
# Inputs: grad_output, color_cache, ray_origins, ray_dirs, grid, opt, log_transmit, accum
# Outputs: grads

def render_ray_backward_kernel(grad_output, color_cache, ray_origins, ray_dirs, grid, opt, log_transmit, accum):
    # 1. Thread ID
    global_tid = get_global_thread_id()
    ray_id = global_tid // grid.sh_data_dim
    ray_blk_id = global_tid // (grid.sh_data_dim * WARP_SIZE)
    lane_id = global_tid % WARP_SIZE
    
    # Early exit if lane out of bounds
    if lane_id >= grid.sh_data_dim:
        return

    # 2. Shared Memory Init
    # sphfunc_val, grad_sphfunc_val, ray_spec, temp_storage allocated in shared memory
    
    # 3. Ray Setup
    ray_spec.origin = ray_origins[ray_id]
    ray_spec.dir = ray_dirs[ray_id]
    vdir = normalize(ray_spec.dir)
    
    # 4. Init Grad Buffers
    if lane_id < grid.basis_dim:
        grad_sphfunc_val[lane_id] = 0.0f
        
    # 5. SH Evaluation
    # Assumes current ray position is available from forward pass or re-evaluated
    sphfunc_val = calc_sphfunc(vdir, grid.basis_dim)
    
    # 6. Ray Bounds (Lane 0 only)
    if lane_id == 0:
        ray_bounds = ray_find_bounds(ray_spec, grid)
        
    # 7. Gradient Normalization
    if grad_out_is_rgb:
        resid = color_cache[ray_id] - grad_output[ray_id]
        norm_factor = 2.0f / (3.0f * Q)
        grad_out = resid * norm_factor
    else:
        grad_out = grad_output[ray_id]
        
    # 8. Warp Sync
    sync_warps_in_sh_mask()
    
    # 9. Volume Backward
    trace_ray_cuvol_backward(
        grad_out=grad_out,
        color_cache=color_cache[ray_id],
        ray_spec=ray_spec,
        opt=opt,
        lane_id=lane_id,
        sphfunc_val=sphfunc_val,
        grad_sphfunc_val=grad_sphfunc_val,
        temp_storage=temp_storage,
        log_transmit_in=log_transmit[ray_id],
        beta_loss=opt.beta_loss,
        sparsity_loss=opt.sparsity_loss,
        grads=grads,
        accum=accum[ray_id] if accum is not None else None,
        log_transmit_out=log_transmit[ray_id]
    )
    
    # 10. SH Backward
    calc_sphfunc_backward(
        grad_sphfunc_val=grad_sphfunc_val,
        vdir=vdir,
        basis_dim=grid.basis_dim,
        grads_sh=grads.sh_coeffs
    )
```

## Subtopics

### Background Rendering Backward Pass
Computes gradients specifically for the background rendering component. It handles RGB gradient normalization (using the same `norm_factor = 2.0f / (3.0f * Q)` logic if `grad_out_is_rgb`) and calls `render_background_backward` [S1, unofficial]. It checks if `log_transmit[ray_id] < -25.0f` and returns early if the ray is fully opaque/terminated, avoiding unnecessary computation [S1, unofficial]. It requires `SingleRaySpec`, background bounds, and the accumulated background color `accum[ray_id]` [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays and the SH basis dimension. The backward pass mirrors the forward pass complexity, dominated by the `trace_ray_cuvol_backward` and `calc_sphfunc_backward` calls [S1, unofficial].
*   **Memory Complexity:** Requires shared memory for `sphfunc_val` and `grad_sphfunc_val` proportional to the SH basis dimension. Global memory access patterns are coalesced for ray origins/directions and gradients [S1, unofficial].
*   **Performance:** Optimized for CUDA with warp-level reductions. Performance scales with GPU occupancy and memory bandwidth [S1, unofficial].
*   **Strengths:** Enables end-to-end gradient flow for SH-based volume rendering. Integrates regularization losses (beta, sparsity) directly into the backward pass [S1, unofficial].
*   **Limitations:** High memory usage for large SH bases. Precision issues may arise with very small transmittance values (handled by the `-25.0f` threshold in background pass) [S1, unofficial].

## Background
The Ray Tracing Backward Pass is part of the svox2 library, developed by Alex Yu and collaborators [S1, unofficial]. It is designed to support differentiable volumetric rendering using Spherical Harmonics for basis functions, allowing for high-quality novel view synthesis and 3D reconstruction [S1, unofficial]. The implementation leverages CUDA for parallel execution on GPUs [S1, unofficial].

## Sources
[S1] svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
