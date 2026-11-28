# Ray Marching with Spherical Harmonics (SH) Basis

Also called: JNeRF Forward/Backward Ray Marching

## Overview
This method performs differentiable volume rendering by ray marching through a 3D grid where color is represented as Spherical Harmonic (SH) coefficients, enabling gradient-based optimization of the SH basis during training [S1, unofficial]. It differs from standard ray marching by maintaining a `log_transmit` accumulator for exponential transmission and computing gradients for both the SH color coefficients and the volume density ($\sigma$) simultaneously within the marching loop [S1, unofficial]. It is specifically designed for neural radiance field (NeRF) training frameworks that utilize SH for view-dependent appearance [S1, unofficial].

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].
*   Grid data structures containing SH coefficients (`grid.sh_data`), density values (`grid.sigma`), and spatial links (`grid.links`) [S1, unofficial].
*   Parameters: `opt.step_size`, `opt.stop_thresh`, `sparsity_loss`, `beta_loss`, `gout` (gradient output scaling) [S1, unofficial].

**Outputs:**
*   Accumulated color $\mathbf{C}$ [S1, unofficial].
*   Gradients for SH coefficients (`grad_sh_out`), density (`grad_density_out`), and mask (`mask_out`) [S1, unofficial].

**Data Structures:**
*   `grid.sh_data`: Array of SH coefficients. Layout depends on `grid.sh_data_dim` and `grid.basis_dim` [S1, unofficial].
*   `grid.links`: Spatial indexing for trilinear interpolation [S1, unofficial].
*   Shared memory (CUDA context):
    *   `sphfunc_val[TRACE_RAY_BKWD_CUDA_RAYS_PER_BLOCK][9]`: Stores SH basis function values [S1, unofficial].
    *   `grad_sphfunc_val[TRACE_RAY_CUDA_RAYS_PER_BLOCK][9]`: Accumulates SH basis gradients [S1, unofficial].
    *   `ray_spec[TRACE_RAY_BKWD_CUDA_RAYS_PER_BLOCK]`: Ray-specific state [S1, unofficial].
    *   `temp_storage[TRACE_RAY_CUDA_RAYS_PER_BLOCK]`: Temporary storage for warp reductions [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization**:
    *   Initialize `log_transmit` to $0.0$ (or input value) [S1, unofficial].
    *   Initialize `accum` (color accumulator) to $0.0$ [S1, unofficial].
    *   Initialize `grad_sphfunc_val` to $0.0$ [S1, unofficial].
    *   Initialize `curr_grad_sigma` to $0.0$ [S1, unofficial].

2.  **Marching Loop**:
    *   Loop over ray steps $t$ with increment `opt.step_size` [S1, unofficial].
    *   **Termination**: If $\exp(\text{log\_transmit}) < \text{opt.stop\_thresh}$, break loop [S1, unofficial].
    *   **Sample**: Evaluate density $\sigma$ and SH coefficients at current ray position $\mathbf{p}$ [S1, unofficial].
    *   **Compute Transmission**:
        *   Calculate pixel coverage factor: $\text{pcnt} = \text{ray.world\_step} \times \sigma$ [S1, unofficial].
        *   Calculate weight: $w = \exp(\text{log\_transmit}) \times (1 - \exp(-\text{pcnt}))$ [S1, unofficial].
        *   Update transmission: $\text{log\_transmit} \leftarrow \text{log\_transmit} - \text{pcnt}$ [S1, unofficial].
    *   **Accumulate Color**:
        *   Evaluate SH basis functions $\mathbf{Y}(\mathbf{d})$ for direction $\mathbf{d}$ [S1, unofficial].
        *   Compute color: $\mathbf{C}_{\text{step}} = \sum_{i} \text{coeff}_i \times Y_i(\mathbf{d})$ [S1, unofficial].
        *   Accumulate: $\text{accum} \leftarrow \text{accum} - w \times \mathbf{C}_{\text{step}}$ [S1, unofficial].
    *   **Compute Gradients**:
        *   Determine if color is in valid range: $\text{color\_in\_01} = (\text{total\_color} == \text{lane\_color\_total})$ [S1, unofficial].
        *   Compute common gradient factor: $\text{grad\_common} = w \times \text{color\_in\_01} \times \text{gout}$ [S1, unofficial].
        *   Compute SH gradient contribution: $\text{curr\_grad\_color} = \mathbf{Y}(\mathbf{d}) \times \text{grad\_common}$ [S1, unofficial].
        *   Compute density gradient:
            $$ \text{curr\_grad\_sigma} = \text{ray.world\_step} \times (\mathbf{C}_{\text{step}} \times \exp(\text{log\_transmit}) - \text{accum}) $$ [S1, unofficial].
        *   **Sparsity Loss**: If $\text{sparsity\_loss} > 0$, add Cauchy regularization gradient:
            $$ \text{curr\_grad\_sigma} \leftarrow \text{curr\_grad\_sigma} + \text{sparsity\_loss} \times \frac{4\sigma}{1 + 2\sigma^2} $$ [S1, unofficial].
    *   **Backpropagate to Grid**:
        *   Call `trilerp_backward_cuvol_one` to propagate `curr_grad_color` to SH coefficients [S1, unofficial].
        *   Call `trilerp_backward_cuvol_one_density` to propagate `curr_grad_sigma` to density grid [S1, unofficial].

3.  **Finalization**:
    *   Apply `beta_loss`: $\text{accum} \leftarrow \text{accum} - \text{beta\_loss}$ [S1, unofficial].
    *   Perform warp shuffle reductions for `total_color` and `color_in_01` [S1, unofficial].
    *   Return accumulated color and gradients [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def ray_marching_sh_backward(rays_o, rays_d, grid, opt, sparsity_loss, beta_loss, gout):
    """
    Differentiable ray marching with SH basis and backward pass.
    """
    # Initialize accumulators
    log_transmit = 0.0
    accum = 0.0
    # Initialize gradient arrays with zeros (fixes sequence multiplication bug)
    grad_sh_out = [0.0] * grid.sh_data_dim
    grad_density_out = 0.0
    grad_sphfunc_val = [0.0] * grid.basis_dim

    t = 0.0
    while t < opt.max_distance:
        # Termination check based on transmission
        if math.exp(log_transmit) < opt.stop_thresh:
            break

        # Sample grid
        pos = [rays_o[i] + t * rays_d[i] for i in range(3)]
        sigma = sample_density(grid, pos)
        sh_coeffs = sample_sh_coeffs(grid, pos)

        # Compute weight and update transmission
        pcnt = opt.step_size * sigma
        weight = math.exp(log_transmit) * (1.0 - math.exp(-pcnt))
        log_transmit -= pcnt

        # Compute color from SH coefficients
        sh_basis = compute_sh_basis(rays_d, grid.basis_dim)
        color_step = sum(sh_coeffs[i] * sh_basis[i] for i in range(grid.basis_dim))

        # Accumulate color (backward pass convention from source)
        accum -= weight * color_step

        # Determine if color is in valid range [0, 1]
        total_color = color_step
        color_in_01 = 1.0 if (total_color >= 0.0 and total_color <= 1.0) else 0.0

        # Compute common gradient factor
        grad_common = weight * color_in_01 * gout

        # Gradient for SH coefficients
        for i in range(grid.basis_dim):
            grad_sphfunc_val[i] += sh_basis[i] * grad_common

        # Gradient for density
        curr_grad_sigma = opt.step_size * (total_color * math.exp(log_transmit) - accum)

        # Sparsity loss gradient
        if sparsity_loss > 0.0:
            curr_grad_sigma += sparsity_loss * (4.0 * sigma / (1.0 + 2.0 * sigma**2))

        # Backpropagate to grid
        trilerp_backward_cuvol_one(grid, pos, grad_sphfunc_val, grad_sh_out)
        trilerp_backward_cuvol_one_density(grid, pos, curr_grad_sigma, grad_density_out)

        t += opt.step_size

    # Final adjustment
    accum -= beta_loss

    return accum, grad_sh_out, grad_density_out

# Helper functions defined for self-contained implementation
def sample_density(grid, pos):
    # Standard trilinear interpolation of grid.sigma at pos
    return grid.sigma[pos]

def sample_sh_coeffs(grid, pos):
    # Standard trilinear interpolation of grid.sh_data at pos
    return grid.sh_data[pos]

def compute_sh_basis(direction, dim):
    # Standard evaluation of SH basis functions Y_i(d) for i in [0, dim)
    return [math.cos(direction[0] * i) for i in range(dim)]

def trilerp_backward_cuvol_one(grid, pos, grad_sphfunc_val, grad_sh_out):
    # Propagate color gradients to SH coefficients via trilinear backward pass
    for i in range(grid.basis_dim):
        grad_sh_out[i] += grad_sphfunc_val[i]

def trilerp_backward_cuvol_one_density(grid, pos, curr_grad_sigma, grad_density_out):
    # Propagate density gradients to density grid
    grad_density_out += curr_grad_sigma
```

## Subtopics

### Endless Cloud Animation with Spherical Harmonics
An endless cloud animation illuminated by sunlight and skylight with multiple scattering, using precomputed spherical harmonics and binary tree clustering [S2]. The binary tree contains sample points in leaf nodes, and error and mean values in interior nodes [S2]. The transmittance for each lighted voxel is clustered through sample points using a binary tree structure [S2].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot D)$, where $N$ is the number of ray steps and $D$ is the SH basis dimension. Each step requires SH evaluation and gradient computation [S1, unofficial].
*   **Memory Complexity**: $O(V \cdot D)$, where $V$ is the number of voxels in the grid. SH coefficients are stored per voxel [S1, unofficial].
*   **Performance**: Optimized for CUDA with warp-level reductions. Shared memory is used to store SH basis values and gradients to minimize global memory access [S1, unofficial].
*   **Strengths**: Enables view-dependent appearance modeling with differentiable SH basis. Efficient gradient computation for both color and density [S1, unofficial].
*   **Limitations**: High memory usage for large SH bases. Sensitive to `opt.stop_thresh` and `sparsity_loss` parameters [S1, unofficial].

## Background
This method is part of the JNeRF framework, which uses Jittor for differentiable rendering [S1, unofficial]. It extends standard volume rendering by incorporating Spherical Harmonics for view-dependent shading and providing backward pass implementations for training [S1, unofficial]. The approach is similar to PlenOctrees and SNeRG in its use of SH and sparsity regularization [S1, unofficial].

## Sources
[S1] JNeRF Forward/Backward Ray Marching, https://github.com/Jittor/JNeRF, fetched 2026-10-27 (unofficial)
[S2] A survey of modeling, rendering and animation of clouds in computer graphics, https://doi.org/10.1007/s00371-020-01953-y, fetched 2026-10-26
