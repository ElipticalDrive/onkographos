# Ray Tracing Backward Pass

Also called: Plenoxel Ray Tracing (Backward)

## Overview
The Ray Tracing Backward Pass computes the gradients of a differentiable volume rendering loss with respect to the scene representation (spherical harmonic coefficients and density) and auxiliary parameters, given the rendered image and ground truth. It serves as the differentiable counterpart to the forward ray casting pass, enabling end-to-end training of volumetric models like Plenoxels. Unlike the forward pass which accumulates color and transmittance, this method propagates error signals backward through the ray integration, handling RGB normalization and optional regularization losses (beta, sparsity) [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- `grad_output`: Gradient of the loss with respect to the rendered RGB color (shape: $[N, 3]$ or flattened).
- `color_cache`: Cached accumulated color components from the forward pass (shape: $[N, 3]$ or similar structure storing intermediate accumulation terms).
- `ray_spec`: Ray origins and directions.
- `grid`: Volumetric grid containing density data and spherical harmonic coefficients.
- `beta_loss`: Scalar weight for beta distribution sparsity loss.
- `sparsity_loss`: Scalar weight for sparsity loss.
- `log_transmit_in`: Logarithm of transmittance at the start of the ray segment.
- `opt`: Configuration options (e.g., `grad_out_is_rgb`, `basis_dim`, `sh_data_dim`).

**Outputs:**
- `grads`: Gradients for spherical harmonic coefficients and density.
- `accum_out` (optional): Accumulated gradient values.
- `log_transmit_out` (optional): Gradient of log transmittance.

**Data Structures:**
- `sphfunc_val`: Shared memory array storing spherical harmonic function values for the current ray.
- `grad_sphfunc_val`: Shared memory array storing gradients of SH function values.
- `ray_spec`: Struct containing ray origin ($\mathbf{o}$) and direction ($\mathbf{v}$).
- `temp_storage`: Warp-level reduction buffer.

**Steps:**

1. **Initialize Accumulation:**
   Compute the initial gradient accumulator `accum` from the color cache and output gradient. Assuming `color_cache` and `grad_output` are vectors of size 3 (RGB):
   $$ \text{accum} = \sum_{i=0}^{2} \text{color\_cache}[i] \cdot \text{grad\_output}[i] $$
   [S1, unofficial]

2. **Thread Identification:**
   Calculate global thread ID, `ray_id`, `ray_blk_id`, and `lane_id` (warp lane index). If `lane_id >= grid.sh_data_dim`, return early, as this thread does not contribute to SH gradient computation.
   [S2, unofficial]

3. **Beta Loss Gradient (if applicable):**
   If `beta_loss > 0`, compute the transmittance at the input:
   $$ t_{\text{in}} = \exp(\log\_transmit\_in) $$
   Update the beta loss contribution to the accumulator:
   $$ \text{beta\_loss\_grad} = \text{beta\_loss} \cdot \left( 1 - \frac{t_{\text{in}}}{1 - t_{\text{in}} + \epsilon} \right) $$
   where $\epsilon = 10^{-3}$ for numerical stability. Add this to `accum`.
   [S1, unofficial]

4. **Shared Memory and Ray Setup:**
   Initialize shared memory arrays `sphfunc_val`, `grad_sphfunc_val`, and `temp_storage`. Set `ray_spec` from origins and directions, extracting the direction vector $\mathbf{v}_{\text{dir}}$.
   [S2, unofficial]

5. **Degenerate Ray Check:**
   If `ray.tmin > ray.tmax`, the ray is invalid. Write `accum` to `*accum_out` (if not null) and `0` to `*log_transmit_out` (if not null), then return.
   [S1, unofficial]

6. **Gradient Output Normalization:**
   If `grad_out_is_rgb` is true, calculate the residual between the cached color and the gradient output:
   $$ \text{resid} = \text{color\_cache} - \text{grad\_output} $$
   Compute normalization factor:
   $$ \text{norm\_factor} = \frac{2.0}{3 \cdot Q} $$
   where $Q$ is the number of rays or a scaling factor defined by the implementation. Update `grad_out[i]`:
   $$ \text{grad\_out}[i] = \text{resid}[i] \cdot \text{norm\_factor} $$
   Otherwise, `grad_out[i] = grad_output[i]`.
   [S2, unofficial]

7. **Warp Synchronization:**
   Synchronize warps within the SH data dimension mask to ensure shared memory consistency.
   [S2, unofficial]

8. **Core Backward Trace:**
   Call `trace_ray_cuvol_backward` with arguments: `grad_out`, `color_cache`, `ray_spec`, `opt`, `lane_id`, `sphfunc_val`, `grad_sphfunc_val`, `temp_storage`, `log_transmit_in`, `beta_loss`, `sparsity_loss`, `grads`, and optional output pointers. This function performs the core integration of gradients along the ray, interpolating density and SH coefficients, and computing gradients for the grid parameters.
   [S2, unofficial]

9. **SH Gradient Computation:**
   Call `calc_sphfunc_backward` to compute the gradients for the spherical harmonic basis coefficients based on the accumulated gradients in `grad_sphfunc_val`.
   [S2, unofficial]

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray_backward_kernel(
    grad_output,      # Tensor [N, 3] or flattened
    color_cache,      # Tensor [N, 3] or similar
    ray_spec,         # Struct with origin, direction, tmin, tmax
    grid,             # Object with density_data, sh_coeffs, sh_data_dim, basis_dim
    beta_loss,        # Float
    sparsity_loss,    # Float
    log_transmit_in,  # Float
    opt,              # Struct with grad_out_is_rgb, sh_data_dim, etc.
    accum_out,        # Optional pointer/tensor
    log_transmit_out  # Optional pointer/tensor
):
    # 1. Initialize Accumulation
    accum = 0.0
    for i in range(3):
        accum += color_cache[i] * grad_output[i]

    # 2. Thread Identification
    lane_id = get_lane_id()
    if lane_id >= grid.sh_data_dim:
        return

    # 3. Beta Loss Gradient
    if beta_loss > 0.0:
        transmit_in = exp(log_transmit_in)
        beta_grad = beta_loss * (1.0 - transmit_in / (1.0 - transmit_in + 1e-3))
        accum += beta_grad

    # 4. Shared Memory Init
    sphfunc_val = shared_mem_array(size=grid.basis_dim)
    grad_sphfunc_val = shared_mem_array(size=grid.basis_dim, init=0.0)
    temp_storage = warp_reduction_buffer()

    # 5. Ray Setup
    vdir = ray_spec.direction
    ray_spec.origin = ray_spec.origin
    ray_spec.direction = vdir

    # 6. Degenerate Ray Check
    if ray_spec.tmin > ray_spec.tmax:
        if accum_out is not None:
            accum_out[0] = accum
        if log_transmit_out is not None:
            log_transmit_out[0] = 0.0
        return

    # 7. Gradient Output Normalization
    if opt.grad_out_is_rgb:
        resid = color_cache - grad_output
        norm_factor = 2.0 / (3.0 * Q) # Q is implementation-specific scaling factor
        grad_out = resid * norm_factor
    else:
        grad_out = grad_output

    # 8. Warp Sync
    sync_warps()

    # 9. Core Backward Trace
    # This step involves trilinear interpolation of density and SH values,
    # and accumulation of gradients along the ray.
    # The exact implementation of trace_ray_cuvol_backward is complex and
    # relies on the forward pass cache.
    trace_ray_cuvol_backward(
        grad_out=grad_out,
        color_cache=color_cache,
        ray_spec=ray_spec,
        opt=opt,
        lane_id=lane_id,
        sphfunc_val=sphfunc_val,
        grad_sphfunc_val=grad_sphfunc_val,
        temp_storage=temp_storage,
        log_transmit_in=log_transmit_in,
        beta_loss=beta_loss,
        sparsity_loss=sparsity_loss,
        grads=grid.grads,
        accum_out=accum_out,
        log_transmit_out=log_transmit_out
    )

    # 10. SH Gradient Computation
    calc_sphfunc_backward(
        grad_sphfunc_val=grad_sphfunc_val,
        sh_coeffs=grid.sh_coeffs,
        grads=grid.grads
    )
```

**Parameters:**
- `beta_loss`: Scalar weight for beta distribution sparsity loss. If $> 0$, gradient contribution is added to `accum` [S1, unofficial][S2, unofficial].
- `sparsity_loss`: Scalar weight for sparsity loss. Passed to `trace_ray_cuvol_backward` [S1, unofficial][S2, unofficial].
- `Q`: Scaling factor used in RGB normalization. Not explicitly defined in sources, but used in formula [S2, unofficial].

**Edge Cases:**
- If `ray.tmin > ray.tmax`, the kernel returns early, writing zero to `log_transmit_out` [S1, unofficial].
- Numerical stability is handled by adding $\epsilon = 10^{-3}$ in the beta loss denominator [S1, unofficial].

## Subtopics

### [Background Rendering Backward Pass](5974-background-rendering-backward-pass.md)
The Background Rendering Backward Pass computes the gradients of the loss with respect to the background volumetric field (typically a sparse grid of spherical harmonic coefficients or density values) and the accumulated transmittance, given the rendered background color, ground truth, and ray parameters. It serves as the differentiable counterpart to the concentric sphere sampling and trilinear interpolation used in the forward background rendering pass. Unlike the foreground ray casting which traverses a dense or sparse voxel grid, this method specifically handles the unbounded background field, often using a separate sparse grid structure. It is distinct from the general Ray Tracing Backward Pass by isolating the gradient computation for the background component, allowing for specialized handling of RGB normalization and sparsity regularization specific to the background layer.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of samples per ray and the number of rays. The SH evaluation and gradient computation are $O(D)$ per sample, where $D$ is the SH dimension.
- **Memory Complexity:** $O(D)$ per thread for shared memory storage of SH function values and gradients.
- **Performance:** Implemented as a CUDA kernel for GPU acceleration. Performance depends on the number of rays, SH dimension, and grid resolution.
- **Strengths:** Enables end-to-end training of volumetric representations. Handles complex lighting via SH.
- **Limitations:** High memory usage for large SH dimensions. Numerical stability issues can arise with extreme transmittance values.
- **When to use:** When training differentiable volume rendering models like Plenoxels.
- **When not to use:** For non-differentiable rendering or when SH approximation is insufficient.

## Background
The Ray Tracing Backward Pass is part of the svox2 library, developed by Alex Yu and others. It is designed to support differentiable volume rendering with spherical harmonics, enabling efficient training of neural radiance fields and related representations. The method is integrated into the NeRF-Factory framework by POSTECH, KAIST, and Kakao Brain Corp. [S1, unofficial][S2, unofficial].

## Sources
[S1] kakaobrain/nerf-factory, https://github.com/kakaobrain/nerf-factory, fetched 2026-10-26 (unofficial)
[S2] sxyu/svox2, https://github.com/sxyu/svox2, fetched 2026-10-27 (unofficial)
