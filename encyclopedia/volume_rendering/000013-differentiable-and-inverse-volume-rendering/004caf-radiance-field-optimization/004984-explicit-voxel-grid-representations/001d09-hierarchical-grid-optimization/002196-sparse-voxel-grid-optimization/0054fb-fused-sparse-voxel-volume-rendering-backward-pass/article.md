# Fused Sparse Voxel Volume Rendering Backward Pass


## Overview
The fused sparse voxel volume rendering backward pass computes gradients for spherical harmonics (SH) coefficients and spherical harmonics basis values during differentiable volume rendering [S1, unofficial]. It specifically supports gradient accumulation for beta loss and sparsity loss regularization terms [S1, unofficial]. This method is part of the JNeRF implementation, which utilizes a fused kernel approach to optimize memory bandwidth and compute throughput for sparse voxel grids [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   `color_cache`: A tensor containing pre-computed color contributions from the forward pass, typically structured as a sequence of components (e.g., RGB or SH basis projections).
*   `grad_output`: The upstream gradient tensor with respect to the rendered pixel color.
*   `log_transmit_in`: The logarithmic transmission value at the start of the ray from the forward pass.
*   `beta_loss`: A scalar value indicating the weight of the beta loss regularization.
*   `sparsity_loss`: A scalar value for sparsity loss regularization (present in signature, usage details not fully explicit in snippet) [S1, unofficial].
*   `accum_out`: Output buffer for the accumulated gradient.
*   `log_transmit_out`: Output buffer for the updated log transmission.

**Data Structures:**
*   `PackedGridOutputGrads`: A structure containing gradients for grid outputs, including density and SH coefficients [S1, unofficial].
*   `color_cache`: Layout is not explicitly defined in the source, but accessed via indices 0, 1, and 2, suggesting a packed or chunked representation of color contributions [S1, unofficial].

**Parameters:**
*   `beta_loss`: Scalar. If greater than 0, the beta loss gradient is computed and added to the accumulation [S1, unofficial].
*   `epsilon`: A small constant ($10^{-3}$) used in the beta loss denominator to prevent division by zero [S1, unofficial].

**Steps:**
1.  **Gradient Accumulation Initialization**: Compute the initial gradient accumulation by taking the dot product of the `color_cache` and `grad_output`. The source uses a fused multiply-add (`fmaf`) chain to compute this efficiently:
    $$ \text{accum} = \text{fmaf}(\text{color\_cache}[0], \text{grad\_output}[0], \text{fmaf}(\text{color\_cache}[1], \text{grad\_output}[1], \text{color\_cache}[2] \times \text{grad\_output}[2])) $$
    This effectively calculates $\sum_{i=0}^{2} \text{color\_cache}[i] \times \text{grad\_output}[i]$ [S1, unofficial].

2.  **Beta Loss Gradient**: If `beta_loss` > 0, compute the gradient contribution from the beta loss term:
    *   Compute the linear transmission from the log input: $\text{transmit\_in} = \exp(\text{log\_transmit\_in})$.
    *   Update the `beta_loss` value (which acts as the gradient accumulator for this term) using the formula:
        $$ \text{beta\_loss} \leftarrow \text{beta\_loss} \times \left(1 - \frac{\text{transmit\_in}}{1 - \text{transmit\_in} + 10^{-3}}\right) $$
    *   Add the updated `beta_loss` to the `accum` variable [S1, unofficial].

3.  **Ray Bounds Check**: Check if the ray is invalid or out of bounds:
    *   If $\text{ray}.t_{\text{min}} > \text{ray}.t_{\text{max}}$:
        *   Write `accum` to `accum_out` if `accum_out` is not null.
        *   Write $0$ to `log_transmit_out` if `log_transmit_out` is not null.
        *   Return immediately [S1, unofficial].

4.  **Rematerialization**: The implementation notes the use of rematerialization (`// remat samples`) to recompute forward pass values during the backward pass, likely to save memory at the cost of additional computation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fused_sparse_voxel_volume_rendering_backward(
    color_cache,
    grad_output,
    log_transmit_in,
    beta_loss,
    sparsity_loss,
    ray_tmin,
    ray_tmax,
    accum_out,
    log_transmit_out
):
    # Step 1: Gradient Accumulation Initialization
    # Compute dot product of color_cache and grad_output using fused multiply-add
    # Assuming color_cache and grad_output have at least 3 elements
    term0 = color_cache[0] * grad_output[0]
    term1 = color_cache[1] * grad_output[1]
    term2 = color_cache[2] * grad_output[2]
    
    # Fused multiply-add chain as per source
    accum = term0 + (term1 + term2)
    
    # Step 2: Beta Loss Gradient
    if beta_loss > 0:
        # Compute linear transmission from log transmission
        transmit_in = exp(log_transmit_in)
        
        # Update beta_loss gradient accumulator
        # Formula: beta_loss *= (1 - transmit_in / (1 - transmit_in + epsilon))
        epsilon = 1e-3
        beta_loss_gradient = beta_loss * (1 - transmit_in / (1 - transmit_in + epsilon))
        
        # Add to accumulation
        accum = accum + beta_loss_gradient
    
    # Step 3: Ray Bounds Check
    if ray_tmin > ray_tmax:
        if accum_out is not None:
            accum_out = accum
        if log_transmit_out is not None:
            log_transmit_out = 0.0
        return accum, log_transmit_out
    
    # Step 4: Rematerialization
    # Note: The source indicates rematerialization of samples is used.
    # In a full implementation, forward pass values (density, SH basis, etc.)
    # would be recomputed here to calculate gradients for SH coefficients and density.
    # Specific formulas for SH and density gradients are not provided in the snippet.
    # [G] Typically, this involves backpropagating through the volume rendering integral
    # using the recomputed transmittance and density values.
    
    # Final output assignment (if not returned early)
    if accum_out is not None:
        accum_out = accum
    if log_transmit_out is not None:
        log_transmit_out = log_transmit_in # Or updated value depending on full logic
        
    return accum, log_transmit_out
```

## Operating characteristics
*   **Time Complexity**: Not explicitly stated, but the use of fused operations (`fmaf`) suggests optimized compute time for the accumulation and beta loss steps [S1, unofficial].
*   **Memory Complexity**: The method employs rematerialization to reduce memory footprint by recomputing forward pass values instead of storing them, trading compute for memory [S1, unofficial].
*   **Strengths**: Efficient gradient computation for SH coefficients and beta loss within a fused kernel framework [S1, unofficial].
*   **Limitations**: The specific implementation details for SH coefficient gradients and density gradients are not fully detailed in the provided snippet, relying on the rematerialization comment [S1, unofficial].

## Background
The method is part of the JNeRF project, an unofficial NeRF benchmark implementation based on the Jittor deep learning framework [S1, unofficial]. It is designed to support differentiable volume rendering with sparse voxel grids, enabling optimization of radiance fields with regularization terms like beta loss [S1, unofficial].

## Sources
[S1] Jittor/JNeRF, https://github.com/Jittor/JNeRF, fetched 2026-10-27 (unofficial)
