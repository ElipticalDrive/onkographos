# Ray Marching with Spherical Harmonics (SH) Basis

Also called: JNeRF Forward/Backward Ray Marching

## Overview
This method performs differentiable volume rendering by ray marching through a 3D grid where color is represented as Spherical Harmonic (SH) coefficients, enabling gradient-based optimization of the SH basis during training. It differs from standard ray marching by maintaining a `log_transmit` accumulator for exponential transmission and computing gradients for both the SH color coefficients and the volume density ($\sigma$) simultaneously within the marching loop. It is specifically designed for neural radiance field (NeRF) training frameworks that utilize SH for view-dependent appearance.

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   Grid data structures containing SH coefficients (`grid.sh_data`), density values (`grid.sigma`), and spatial links (`grid.links`).
*   Parameters: `opt.step_size`, `opt.stop_thresh`, `sparsity_loss`, `beta_loss`, `gout` (gradient output scaling).

**Outputs:**
*   Accumulated color $\mathbf{C}$.
*   Gradients for SH coefficients (`grad_sh_out`), density (`grad_density_out`), and mask (`mask_out`).

**Data Structures:**
*   `grid.sh_data`: Array of SH coefficients. Layout depends on `grid.sh_data_dim` and `grid.basis_dim`.
*   `grid.links`: Spatial indexing for trilinear interpolation.
*   Shared memory (CUDA context):
    *   `sphfunc_val[TRACE_RAY_BKWD_CUDA_RAYS_PER_BLOCK][9]`: Stores SH basis function values.
    *   `grad_sphfunc_val[TRACE_RAY_CUDA_RAYS_PER_BLOCK][9]`: Accumulates SH basis gradients.
    *   `ray_spec[TRACE_RAY_BKWD_CUDA_RAYS_PER_BLOCK]`: Ray-specific state.
    *   `temp_storage[TRACE_RAY_CUDA_RAYS_PER_BLOCK]`: Temporary storage for warp reductions.

**Algorithm Steps:**

1.  **Initialization**:
    *   Initialize `log_transmit` to $0.0$ (or input value).
    *   Initialize `accum` (color accumulator) to $0.0$.
    *   Initialize `grad_sphfunc_val` to $0.0$.
    *   Initialize `curr_grad_sigma` to $0.0$.

2.  **Marching Loop**:
    *   Loop over ray steps $t$ with increment `opt.step_size`.
    *   **Termination**: If $\exp(\text{log\_transmit}) < \text{opt.stop\_thresh}$, break loop.
    *   **Sample**: Evaluate density $\sigma$ and SH coefficients at current ray position $\mathbf{p}$.
    *   **Compute Transmission**:
        *   Calculate pixel coverage factor: $\text{pcnt} = \text{ray.world\_step} \times \sigma$.
        *   Calculate weight: $w = \exp(\text{log\_transmit}) \times (1 - \exp(-\text{pcnt}))$.
        *   Update transmission: $\text{log\_transmit} \leftarrow \text{log\_transmit} - \text{pcnt}$.
    *   **Accumulate Color**:
        *   Evaluate SH basis functions $\mathbf{Y}(\mathbf{d})$ for direction $\mathbf{d}$.
        *   Compute color: $\mathbf{C}_{\text{step}} = \sum_{i} \text{coeff}_i \times Y_i(\mathbf{d})$.
        *   Accumulate: $\text{accum} \leftarrow \text{accum} + w \times \mathbf{C}_{\text{step}}$.
    *   **Compute Gradients**:
        *   Determine if color is in valid range: $\text{color\_in\_01} = (\text{total\_color} == \text{lane\_color\_total})$.
        *   Compute common gradient factor: $\text{grad\_common} = w \times \text{color\_in\_01} \times \text{gout}$.
        *   Compute SH gradient contribution: $\text{curr\_grad\_color} = \mathbf{Y}(\mathbf{d}) \times \text{grad\_common}$.
        *   Compute density gradient:
            $$ \text{curr\_grad\_sigma} = \text{ray.world\_step} \times (\mathbf{C}_{\text{step}} \times \exp(\text{log\_transmit}) - \text{accum}) $$
        *   **Sparsity Loss**: If $\text{sparsity\_loss} > 0$, add Cauchy regularization gradient:
            $$ \text{curr\_grad\_sigma} \leftarrow \text{curr\_grad\_sigma} + \text{sparsity\_loss} \times \frac{4\sigma}{1 + 2\sigma^2} $$
    *   **Backpropagate to Grid**:
        *   Call `trilerp_backward_cuvol_one` to propagate `curr_grad_color` to SH coefficients.
        *   Call `trilerp_backward_cuvol_one_density` to propagate `curr_grad_sigma` to density grid.

3.  **Finalization**:
    *   Apply `beta_loss`: $\text{accum} \leftarrow \text{accum} - \text{beta\_loss}$.
    *   Perform warp shuffle reductions for `total_color` and `color_in_01`.
    *   Return accumulated color and gradients.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_marching_sh_backward(rays_o, rays_d, grid, opt, sparsity_loss, beta_loss, gout):
    """
    Differentiable ray marching with SH basis and backward pass.
    """
    # Initialize accumulators
    log_transmit = 0.0
    accum = 0.0
    grad_sh_out = zeros_like(grid.sh_data)
    grad_density_out = zeros_like(grid.sigma)
    
    # Precompute SH basis values for direction
    # Note: In implementation, this is done per-step or cached per ray
    sh_basis = compute_sh_basis(rays_d, grid.basis_dim)
    
    t = 0.0
    while t < opt.max_distance:
        # Termination check based on transmission
        if exp(log_transmit) < opt.stop_thresh:
            break
            
        # Sample grid
        pos = rays_o + t * rays_d
        sigma = sample_density(grid, pos)
        sh_coeffs = sample_sh_coeffs(grid, pos)
        
        # Compute weight and update transmission
        pcnt = opt.step_size * sigma
        weight = exp(log_transmit) * (1 - exp(-pcnt))
        log_transmit -= pcnt
        
        # Compute color from SH coefficients
        # C_step = sum(sh_coeffs[i] * sh_basis[i])
        color_step = dot(sh_coeffs, sh_basis)
        
        # Accumulate color
        accum += weight * color_step
        
        # Compute gradients
        # Determine if color is in valid range [0, 1]
        # In CUDA, this is done via warp reductions
        total_color = accum # Simplified for pseudocode
        color_in_01 = (total_color >= 0.0) and (total_color <= 1.0)
        
        grad_common = weight * color_in_01 * gout
        
        # Gradient for SH coefficients
        # grad_sh = sh_basis * grad_common
        grad_sh = sh_basis * grad_common
        
        # Gradient for density
        # curr_grad_sigma = step_size * (color_step * exp(log_transmit) - accum)
        curr_grad_sigma = opt.step_size * (color_step * exp(log_transmit) - accum)
        
        # Sparsity loss gradient
        if sparsity_loss > 0.0:
            curr_grad_sigma += sparsity_loss * (4 * sigma / (1 + 2 * sigma**2))
        
        # Backpropagate to grid (simplified)
        # trilerp_backward_cuvol_one(grid, pos, grad_sh, grad_sh_out)
        # trilerp_backward_cuvol_one_density(grid, pos, curr_grad_sigma, grad_density_out)
        
        t += opt.step_size
        
    # Final adjustment
    accum -= beta_loss
    
    return accum, grad_sh_out, grad_density_out
```

## Operating characteristics
*   **Time Complexity**: $O(N \cdot D)$, where $N$ is the number of ray steps and $D$ is the SH basis dimension. Each step requires SH evaluation and gradient computation.
*   **Memory Complexity**: $O(V \cdot D)$, where $V$ is the number of voxels in the grid. SH coefficients are stored per voxel.
*   **Performance**: Optimized for CUDA with warp-level reductions. Shared memory is used to store SH basis values and gradients to minimize global memory access.
*   **Strengths**: Enables view-dependent appearance modeling with differentiable SH basis. Efficient gradient computation for both color and density.
*   **Limitations**: High memory usage for large SH bases. Sensitive to `opt.stop_thresh` and `sparsity_loss` parameters.

## Background
This method is part of the JNeRF framework, which uses Jittor for differentiable rendering. It extends standard volume rendering by incorporating Spherical Harmonics for view-dependent shading and providing backward pass implementations for training. The approach is similar to PlenOctrees and SNeRG in its use of SH and sparsity regularization.

## Sources
[S1] Jittor/JNeRF, https://github.com/Jittor/JNeRF, fetched 2026-10-27 (unofficial)
