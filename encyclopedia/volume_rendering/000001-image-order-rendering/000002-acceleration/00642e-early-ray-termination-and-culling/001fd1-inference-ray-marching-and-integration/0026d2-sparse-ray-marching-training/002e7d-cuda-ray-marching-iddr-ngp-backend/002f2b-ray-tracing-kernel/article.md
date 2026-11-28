# Ray Tracing Kernel
Also called: CUDA Ray Tracing Kernel (Backward)

## Overview
The Ray Tracing Kernel is a CUDA compute kernel that performs forward and backward volumetric ray marching through a sparse hashed density grid, handling spherical harmonics (SH) evaluation and ray bounds [S2, unofficial]. It serves as the core inference and training step for the IDDR-NGP backend, computing final pixel colors and log-transmittance in the forward pass, and gradients for SH coefficients and grid outputs in the backward pass [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Forward:** Ray origins, directions, grid data, and options. Outputs: pixel color and log-transmittance [S1, unofficial][S2, unofficial].
*   **Backward:** `grad_output`, `color_cache`, `log_transmit_in`, `beta_loss`, `sparsity_loss`, and `grads`. Outputs: gradients for SH coefficients and grid data [S1, unofficial][S2, unofficial].

**Data Structures:**
*   `ray_spec[TRACE_RAY_CUDA_RAYS_PER_BLOCK]`: Shared memory array of `SingleRaySpec` structs storing ray origin and direction for each ray in the block [S1, unofficial][S2, unofficial].
*   `sphfunc_val[TRACE_RAY_CUDA_RAYS_PER_BLOCK][9]`: Shared memory 2D float array storing spherical harmonic function values [S1, unofficial][S2, unofficial].
*   `temp_storage[TRACE_RAY_CUDA_RAYS_PER_BLOCK]`: Shared memory for warp reduction operations [S1, unofficial][S2, unofficial].
*   `grad_sphfunc_val[TRACE_RAY_CUDA_RAYS_PER_BLOCK][9]`: Shared memory 2D float array storing gradients for spherical harmonic values [S1, unofficial].

**Parameters:**
*   `TRACE_RAY_CUDA_RAYS_PER_BLOCK`: Number of rays processed per CUDA block [S1, unofficial].
*   `TRACE_RAY_BKWD_CUDA_THREADS`: Number of threads per block for the backward kernel [S1, unofficial].
*   `grid.basis_dim`: Number of SH basis functions per color channel [S1, unofficial].
*   `grid.sh_data_dim`: Dimension of SH data, used to mask threads [S2, unofficial].

**Algorithm (Forward):**
1.  Calculate thread indices: `ray_blk_id = thread_idx >> 5`, `lane_id = thread_idx & 0x1F` [S1, unofficial][S2, unofficial].
2.  If `lane_id >= grid.sh_data_dim`, return early [S1, unofficial][S2, unofficial].
3.  Initialize `ray_spec[ray_blk_id]` with origin and direction from the rays array for the current `ray_id` [S1, unofficial].
4.  Calculate spherical harmonic function values: `calc_sphfunc(grid, lane_id, ray_id, ray_spec[ray_blk_id].dir, sphfunc_val[ray_blk_id])` [S1, unofficial][S2, unofficial].
5.  Synchronize warps within the active mask defined by `grid.sh_data_dim` to ensure shared memory writes are visible [S1, unofficial][S2, unofficial].
6.  Find ray bounds within the grid: `ray_find_bounds(ray_spec[ray_blk_id], grid, opt, ray_id)` [S1, unofficial][S2, unofficial].
7.  Trace the ray through the volume: `trace_ray_cuvol(grid, ray_spec, opt, lane_id, sphfunc_val, temp_storage, output, log_transmit)` [S1, unofficial][S2, unofficial].

**Algorithm (Backward):**
1.  Initialize gradient SH values to `0.0f` for all lanes less than `grid.basis_dim` [S1, unofficial].
2.  Calculate spherical harmonic function values for the backward pass [S1, unofficial].
3.  Find ray bounds (performed only by lane 0 to avoid redundant computation) [S1, unofficial].
4.  Compute gradient output: If `grad_out_is_rgb` is true, calculate `resid = color_cache - grad_output`, then `grad_out[i] = resid * norm_factor` where `norm_factor = 2.0 / (3 * Q)` [S1, unofficial]. Otherwise, `grad_out[i] = grad_output[i]` [S1, unofficial].
5.  Synchronize warps within the active mask [S1, unofficial].
6.  Trace ray backward: `trace_ray_cuvol_backward(grid, grad_out, color_cache, ray_spec, opt, lane_id, sphfunc_val, grad_sphfunc_val, temp_storage, log_transmit_in, beta_loss, sparsity_loss, grads, accum_out, log_transmit_out)` [S1, unofficial].
7.  Calculate SH gradients: `calc_sphfunc_backward(grid, lane_id, ray_id, vdir, sphfunc_val, grad_sphfunc_val, grads.grad_basis_out)` [S1, unofficial].

**Pseudocode (Forward):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sync_warp(mask):
    # CUDA intrinsic: __syncwarp(mask)
    pass

def render_ray_kernel(thread_idx, rays, grid, opt, out, out_log_transmit):
    ray_blk_id = thread_idx >> 5
    lane_id = thread_idx & 0x1F

    if lane_id >= grid.sh_data_dim:
        return

    ray_id = (blockIdx.x * blockDim.x + threadIdx.x)  # Standard CUDA indexing
    ray_spec[ray_blk_id].origin = rays[ray_id].origin
    ray_spec[ray_blk_id].dir = rays[ray_id].dir

    calc_sphfunc(grid, lane_id, ray_id, ray_spec[ray_blk_id].dir, sphfunc_val[ray_blk_id])

    sync_warp((1 << grid.sh_data_dim) - 1)

    ray_find_bounds(ray_spec[ray_blk_id], grid, opt, ray_id)

    trace_ray_cuvol(grid, ray_spec, opt, lane_id, sphfunc_val, temp_storage, out, out_log_transmit)
```

**Edge Cases and Pitfalls:**
*   If `lane_id >= grid.sh_data_dim`, the thread returns early to handle cases where the number of spherical harmonic basis functions is less than the warp size (32) [S1, unofficial][S2, unofficial].
*   Ray bounds calculation is performed only by lane 0 in the backward pass to avoid redundant computation [S1, unofficial].

## Subtopics

### [TracingKernel](13916-tracingkernel.md)
`TracingKernel` is a CUDA compute shader that performs volumetric light transport by stepping through a 3D grid, dynamically adjusting step size and cascade resolution based on distance from the center, and accumulating transmittance (alpha). It is designed for the IDDR-NGP backend, where it serves as the core inference step for sparse ray marching, optimizing performance by coarser sampling in distant regions and finer sampling near the center. It differs from standard uniform ray marching by using a logarithmic expansion factor to vary step sizes and cascade levels, and from dense voxel cone tracing by operating on a sparse grid structure with specific boundary handling.

## Operating characteristics
*   Time and memory complexity: Not given in the sources.
*   Measured performance: Not given in the sources.
*   Scaling: Not given in the sources.
*   Strengths and limitations: Not given in the sources.
*   When to use and when not: Not given in the sources.

## Background
The Ray Tracing Kernel is part of the NeRF-Factory project by POSTECH, KAIST, and Kakao Brain Corp. [S1, unofficial]. It is designed to work with the IDDR-NGP backend, which uses a sparse hashed density grid for efficient volumetric rendering. The kernel supports both forward rendering and backward gradient computation for training neural radiance fields (NeRFs) [S1, unofficial][S2, unofficial].

## Sources
[S1] kakaobrain/nerf-factory, https://github.com/kakaobrain/nerf-factory, fetched 2026-10-26 (unofficial)
[S2] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
