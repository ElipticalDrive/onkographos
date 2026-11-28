# Fused Sparse Voxel Volume Rendering Backward Pass
Also called: Plenoxel Ray Tracing (Backward)

## Overview
The fused sparse voxel volume rendering backward pass computes gradients for spherical harmonics (SH) coefficients, SH basis values, density, and render accumulators during differentiable volume marching through a packed sparse grid [S1, unofficial]. It natively supports gradient accumulation for beta loss and sparsity loss regularization terms [S2, unofficial]. The method fuses ray marching, color interpolation, and gradient computation into a single kernel to minimize memory bandwidth overhead [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   `color_cache`: A 3-element array holding pre-computed color contributions from the forward pass, accessed via indices 0, 1, and 2 [S1, unofficial].
*   `grad_output`: Upstream gradient tensor with respect to the rendered pixel color [S1, unofficial].
*   `log_transmit_in`: Logarithmic transmission value at the start of the ray from the forward pass [S2, unofficial].
*   `beta_loss`: Scalar regularization weight [S1, unofficial].
*   `sparsity_loss`: Scalar regularization weight for sparsity [S2, unofficial].
*   `ray`: Ray structure containing `tmin`, `tmax`, `world_step`, and current position `t` [S1, unofficial].
*   `grid`: Sparse grid structure containing `basis_dim`, `sh_data_dim`, `basis_type`, and `sh_data` [S1, unofficial].
*   `opt`: Optimization flags containing `last_sample_opaque`, `step_size`, and `sigma_thresh` [S1, unofficial].
*   `lane_id`: Current thread/warp lane identifier [S1, unofficial].
*   `accum_out`: Output buffer for the accumulated gradient [S1, unofficial].
*   `log_transmit_out`: Output buffer for the updated log transmission [S1, unofficial].

**Data Structures:**
*   `PackedGridOutputGrads`: Holds gradients for grid outputs, specifically density and SH coefficients [S2, unofficial].
*   `color_cache`: Packed 3-component representation of color contributions [S1, unofficial].

**Parameters:**
*   `beta_loss`: Scalar. If greater than 0, the beta loss gradient is computed and added to the accumulation [S1, unofficial].
*   `epsilon`: Small constant $10^{-3}$ used in the beta loss denominator to prevent division by zero [S1, unofficial].

**Steps:**
1.  **Lane Grouping Initialization**: Compute `lane_colorgrp_id = lane_id % grid.basis_dim` and `lane_colorgrp = lane_id / grid.basis_dim` [S1, unofficial]. Compute `leader_mask = 1U | (1U << grid.basis_dim) | (1U << (2 * grid.basis_dim))` [S1, unofficial].
2.  **Gradient Accumulation Initialization**: Compute the initial `accum` via a fused multiply-add chain: `accum = fmaf(color_cache[0], grad_output[0], fmaf(color_cache[1], grad_output[1], color_cache[2] * grad_output[2]))` [S1, unofficial].
3.  **Beta Loss Gradient**: If `beta_loss > 0`, compute `transmit_in = exp(log_transmit_in)` [S2, unofficial]. Update `beta_loss *= (1 - transmit_in / (1 - transmit_in + 1e-3))` [S1, unofficial]. Add the updated `beta_loss` to `accum` [S1, unofficial].
4.  **Ray Bounds Check**: If `ray.tmin > ray.tmax`, write `accum` to `accum_out` if not null, write `0` to `log_transmit_out` if not null, and return [S1, unofficial].
5.  **Ray Marching Loop**: While `t <= ray.tmax`: compute ray position, clamp, compute lattice `l` and fractional position [S1, unofficial].
6.  **Density Interpolation**: Trilinearly interpolate density `sigma` [S1, unofficial]. If `opt.last_sample_opaque` and `t + opt.step_size > ray.tmax`, set `ray.world_step = 1e9` [S1, unofficial].
7.  **Color Interpolation**: If `sigma > opt.sigma_thresh`, interpolate color from `grid.sh_data` using `lane_id`, multiply by `sphfunc_val[lane_colorgrp_id]` to get `weighted_lane_color` [S1, unofficial].
8.  **Warp Reduction**: Reduce `weighted_lane_color` across the basis dimension using a segmented sum with `lane_colorgrp_id == 0` as the leader flag [S1, unofficial]. Add `0.5f` [S1, unofficial].
9.  **Color Clamping & Scaling**: Compute `total_color = fmaxf(lane_color_total, 0.f)` [S1, unofficial]. Compute `color_in_01 = (total_color == lane_color_total)` [S1, unofficial]. Multiply `total_color` by `gout` [S1, unofficial].
10. **Shuffle & Accumulate**: Shuffle `total_color` to lanes 0 and `2*grid.basis_dim` using `leader_mask`, and accumulate [S1, unofficial].
11. **Gradient Common Computation**: Shuffle `color_in_01` to lane `lane_colorgrp * grid.basis_dim` using mask `(1U << grid.sh_data_dim) - 1` [S1, unofficial]. Compute `grad_common = weight * color_in_01 * gout` [S1, unofficial].
12. **Color Gradient**: Compute `curr_grad_color = sphfunc_val[lane_colorgrp_id] * grad_common` [S1, unofficial].
13. **SH Basis Gradient**: If `grid.basis_type != BASIS_TYPE_SH`, compute `curr_grad_sphfunc = lane_color * grad_common` [S1, unofficial]. Shuffle down by `2*grid.basis_dim` and `grid.basis_dim`, accumulate [S1, unofficial]. If `lane_id < grid.basis_dim`, add to `grad_sphfunc_val[lane_id]` [S1, unofficial].
14. **Accumulator Update**: Update `accum -= weight * total_color` [S1, unofficial].
15. **Sparsity Loss**: If `sparsity_loss > 0.f`, apply the Cauchy version gradient [S1, unofficial].
16. **Advance Ray**: Advance `t` by `opt.step_size` [S1, unofficial].

**Formulas:**
*   `pcnt = ray.world_step * sigma`
*   `weight = exp(log_transmit) * (1 - exp(-pcnt))` [S1, unofficial]
*   `log_transmit -= pcnt`
*   `curr_grad_sigma = ray.world_step * (total_color * exp(log_transmit) - accum)` [S1, unofficial]

**Edge Cases and Pitfalls:**
*   If `ray.tmin > ray.tmax`, the function must immediately write `accum` to `accum_out` (if not null) and `0` to `log_transmit_out` (if not null) before returning [S1, unofficial].
*   The `opt.last_sample_opaque` flag forces `ray.world_step` to $10^9$ when the ray exits the volume, preventing division by zero or incorrect weight scaling at the boundary [S1, unofficial].
*   The beta loss denominator uses $10^{-3}$ to avoid numerical instability when `transmit_in` approaches 1 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def fused_sparse_voxel_volume_rendering_backward(
    color_cache, grad_output, log_transmit_in, beta_loss, sparsity_loss,
    ray_tmin, ray_tmax, ray_t, ray_world_step, ray_step_size, ray_sigma_thresh,
    ray_last_sample_opaque, grid_basis_dim, grid_sh_data_dim, grid_basis_type,
    grid_sh_data, sphfunc_val, lane_id, gout, accum_out, log_transmit_out
):
    # Step 1: Lane grouping
    lane_colorgrp_id = lane_id % grid_basis_dim
    lane_colorgrp = lane_id // grid_basis_dim
    leader_mask = 1 | (1 << grid_basis_dim) | (1 << (2 * grid_basis_dim))

    # Step 2: Gradient accumulation initialization
    accum = color_cache[0] * grad_output[0] + (color_cache[1] * grad_output[1] + color_cache[2] * grad_output[2])

    # Step 3: Beta loss gradient
    if beta_loss > 0:
        transmit_in = math.exp(log_transmit_in)
        beta_loss = beta_loss * (1.0 - transmit_in / (1.0 - transmit_in + 1e-3))
        accum += beta_loss

    # Step 4: Ray bounds check
    if ray_tmin > ray_tmax:
        if accum_out is not None:
            accum_out = accum
        if log_transmit_out is not None:
            log_transmit_out = 0.0
        return accum, log_transmit_out

    # Step 5-16: Ray marching loop
    log_transmit = log_transmit_in
    grad_sphfunc_val = [0.0] * grid_basis_dim

    while ray_t <= ray_tmax:
        # Compute ray position, clamp, lattice l, fractional pos [S1, unofficial]
        # Trilinearly interpolate density sigma [S1, unofficial]
        sigma = 0.0 # Placeholder for trilinear interpolation logic
        if ray_last_sample_opaque and (ray_t + ray_step_size) > ray_tmax:
            ray_world_step = 1e9

        # Step 6: Density threshold check
        if sigma > ray_sigma_thresh:
            # Interpolate color from grid.sh_data using lane_id [S1, unofficial]
            lane_color = 0.0 # Placeholder for SH interpolation
            weighted_lane_color = lane_color * sphfunc_val[lane_colorgrp_id]

            # Step 8: Warp reduction (simulated with list accumulation)
            lane_color_total = weighted_lane_color + 0.5
            total_color = max(lane_color_total, 0.0)
            color_in_01 = 1.0 if (total_color == lane_color_total) else 0.0
            total_color *= gout

            # Step 9-10: Shuffle & accumulate (simulated)
            # Shuffle total_color to lanes 0 and 2*grid_basis_dim using leader_mask
            # Accumulate into total_color

            # Step 11: Gradient common
            pcnt = ray_world_step * sigma
            weight = math.exp(log_transmit) * (1.0 - math.exp(-pcnt))
            log_transmit -= pcnt
            grad_common = weight * color_in_01 * gout

            # Step 12: Color gradient
            curr_grad_color = sphfunc_val[lane_colorgrp_id] * grad_common

            # Step 13: SH basis gradient
            if grid_basis_type != "BASIS_TYPE_SH":
                curr_grad_sphfunc = lane_color * grad_common
                # Shuffle down by 2*grid_basis_dim and grid_basis_dim, accumulate
                if lane_id < grid_basis_dim:
                    grad_sphfunc_val[lane_id] += curr_grad_sphfunc

            # Step 14: Accumulator update
            accum -= weight * total_color

            # Step 15: Sparsity loss
            if sparsity_loss > 0.0:
                # Apply Cauchy version gradient from SNeRG [G]
                pass

        # Step 16: Advance ray
        ray_t += ray_step_size

    # Compute density gradient
    curr_grad_sigma = ray_world_step * (total_color * math.exp(log_transmit) - accum)

    # Final outputs
    if accum_out is not None:
        accum_out = accum
    if log_transmit_out is not None:
        log_transmit_out = log_transmit_in
    return accum, log_transmit_out, grad_sphfunc_val, curr_grad_sigma
```

## Operating characteristics
*   **Time Complexity**: Not explicitly stated in the sources [S1, unofficial].
*   **Memory Complexity**: The method employs rematerialization to recompute forward pass values during the backward pass, trading additional compute for reduced memory footprint [S1, unofficial].
*   **Strengths**: Fuses ray marching, SH interpolation, and gradient computation into a single kernel, optimizing memory bandwidth and compute throughput for sparse voxel grids [S1, unofficial]. Supports direct gradient accumulation for beta loss and sparsity loss [S2, unofficial].
*   **Limitations**: Relies on rematerialization, increasing compute cost compared to cached forward values [S1, unofficial]. Specific implementation details for SH coefficient gradients and density gradients are abstracted behind warp primitives and rematerialization comments [S1, unofficial].

## Background
The method originates from the Plenoxel and ARF-svox2 ecosystems for differentiable volume rendering on hierarchical sparse voxel grids [S1, unofficial]. It is designed to enable optimization of radiance fields with regularization terms like beta loss and sparsity loss [S2, unofficial]. The implementation is utilized in 3D style transfer and radiance field optimization pipelines [S1, unofficial].

## Sources
[S1] OSHMOS/IPRF, https://github.com/OSHMOS/IPRF, fetched 2026-10-27 (unofficial)
[S2] Jittor/JNeRF, https://github.com/Jittor/JNeRF, fetched 2026-10-27 (unofficial)
