# Plenoxel Optimization with TV Regularization

Also called: Chambolle-Pock TV, Plenoxels TV and sparsity regularization, TV regularization, Total Variation Reconstruction

## Overview
Plenoxel Optimization with TV Regularization reconstructs radiance fields from sparse voxel grids by minimizing a composite loss that combines Mean Squared Error (MSE) reconstruction with isotropic Total Variation (TV) regularization [S1]. It solves the visual artifacts and ill-conditioning that arise when optimizing high-dimensional, non-convex radiance fields with standard gradient descent [S1]. Unlike dense grid constraints or neural network smoothing, this method explicitly penalizes the spatial gradients of voxel opacities ($\sigma$) and spherical harmonic (SH) coefficients to enforce piecewise smoothness while preserving sharp boundaries [S1]. It builds on [Plenoxels regularization ablation](901-plenoxels-regularization-ablation.md) by specifying the stochastic sampling strategy, solver configuration, and per-component weight tuning required for implementation [S1].

## How it works
The method optimizes voxel opacities and SH coefficients using the RMSProp optimizer to mitigate ill-conditioning without computing or inverting the Hessian [S1]. The optimization minimizes a total loss composed of a reconstruction term and a TV regularization term [S1].

### Inputs and Outputs
- **Inputs**: Ground truth pixel colors $C(\mathbf{r})$ for a set of rays $\mathcal{R}$ [S1]. An initial sparse voxel grid containing opacity $\sigma$ and SH coefficients [S1]. A scene type identifier (bounded, forward-facing, or 360°) to select TV weights [S1].
- **Outputs**: Optimized voxel opacities and SH coefficients [S1].

### Data Structures
The method maintains a sparse voxel grid where each active voxel $\mathbf{v} = (i, j, k)$ stores a scalar opacity $\sigma_{\mathbf{v}}$ and a vector of SH coefficients $\mathbf{c}_{\mathbf{v}} \in \mathbb{R}^D$ [S1]. Neighbor relationships are handled via implicit integer indexing $(i\pm1, j, k)$, $(i, j\pm1, k)$, and $(i, j, k\pm1)$ [G].

### Formulas
The total loss $\mathcal{L}$ is defined as [S1]:
$$
\mathcal{L} = \mathcal{L}_{recon} + \lambda_{TV} \mathcal{L}_{TV}
$$

The reconstruction loss $\mathcal{L}_{recon}$ is the MSE between rendered colors $\hat{C}(\mathbf{r})$ and ground truth colors $C(\mathbf{r})$ [S1]:
$$
\mathcal{L}_{recon} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \|C(\mathbf{r}) - \hat{C}(\mathbf{r})\|_2^2
$$

The TV regularization loss $\mathcal{L}_{TV}$ is the isotropic total variation computed over a stochastic batch of active voxels $\mathcal{V}$ and SH dimensions $d \in [D]$ [S1]:
$$
\mathcal{L}_{TV} = \frac{1}{|\mathcal{V}|} \sum_{\substack{\mathbf{v} \in \mathcal{V} \\ d \in [D]}} \sqrt{\Delta_x^2(\mathbf{v}, d) + \Delta_y^2(\mathbf{v}, d) + \Delta_z^2(\mathbf{v}, d)}
$$

Here, $\Delta_x^2(\mathbf{v}, d)$ is the squared difference between the $d$-th value in voxel $\mathbf{v}=(i,j,k)$ and its neighbor $(i+1,j,k)$, normalized by the voxel resolution [S1]. The terms $\Delta_y^2(\mathbf{v}, d)$ and $\Delta_z^2(\mathbf{v}, d)$ are computed analogously for the $y$ and $z$ directions [G].

### Parameters
- **TV Weight ($\lambda_{TV}$)**: Fixed per scene type (bounded, forward-facing, 360°) and assigned separately for SH coefficients and opacity $\sigma$ [S1].
- **Optimizer**: RMSProp [S1].
- **Learning Rate**: Fixed across all scenes in the main experiments, but tuned independently for opacity and harmonic coefficients [S1].
- **Stochastic Sampling**: Rays $\mathcal{R}$ and voxels $\mathcal{V}$ are sampled randomly per optimization step to accelerate iteration [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def rmsprop_step(param, grad, state, lr, beta1=0.9, beta2=0.999, eps=1e-8):
    """Standard RMSProp update step."""
    state['ms'] = beta1 * state['ms'] + (1 - beta1) * grad**2
    state['ms_sqrt'] = math.sqrt(state['ms']) + eps
    param -= lr * state['ms'] / state['ms_sqrt']
    return param

def compute_tv(field, voxels, resolution):
    """Computes isotropic TV over a batch of voxels for a single field (sigma or SH channel)."""
    tv_sum = 0.0
    count = 0
    for v in voxels:
        i, j, k = v
        for d in range(field.dim):
            val = field[i, j, k, d]
            val_x = field[i + 1, j, k, d]
            val_y = field[i, j + 1, k, d]
            val_z = field[i, j, k + 1, d]
            dx = (val - val_x) / resolution
            dy = (val - val_y) / resolution
            dz = (val - val_z) / resolution
            tv_sum += math.sqrt(dx**2 + dy**2 + dz**2)
            count += 1
    return tv_sum / count if count > 0 else 0.0

def optimize_plenoxels_tv(voxel_grid, rays, gt_colors, scene_type, n_iters, 
                          lr_sigma, lr_sh, lambda_tv_sigma, lambda_tv_sh, 
                          voxel_resolution, batch_size_rays, batch_size_voxels):
    """Optimizes Plenoxel grid using TV regularization and RMSProp."""
    # Initialize RMSProp state
    state_sigma = {'ms': 0.0}
    state_sh = {'ms': 0.0}
    
    for step in range(n_iters):
        # 1. Stochastic Sampling [S1]
        batch_rays = random.sample(rays, batch_size_rays)
        batch_voxels = random.sample(voxel_grid.active_indices, batch_size_voxels)
        
        # 2. Compute Reconstruction Loss [S1]
        # Note: render_rays computes standard volume rendering for the batch [G]
        rendered_colors = render_rays(voxel_grid, batch_rays)
        mse_sum = sum((gt_colors[r] - rendered_colors[r])**2 for r in batch_rays)
        l_recon = mse_sum / len(batch_rays)
        
        # 3. Compute TV Regularization Loss [S1]
        l_tv_sigma = compute_tv(voxel_grid.opacities, batch_voxels, voxel_resolution)
        l_tv_sh = compute_tv(voxel_grid.sh_coeffs, batch_voxels, voxel_resolution)
        
        # 4. Combine Losses [S1]
        l_tv = lambda_tv_sigma * l_tv_sigma + lambda_tv_sh * l_tv_sh
        total_loss = l_recon + l_tv
        
        # 5. Backpropagation and Update [S1]
        # Gradients are computed via automatic differentiation of total_loss
        grad_sigma = compute_gradient(total_loss, voxel_grid.opacities)
        grad_sh = compute_gradient(total_loss, voxel_grid.sh_coeffs)
        
        # Update using RMSProp [S1]
        voxel_grid.opacities = rmsprop_step(voxel_grid.opacities, grad_sigma, state_sigma, lr_sigma)
        voxel_grid.sh_coeffs = rmsprop_step(voxel_grid.sh_coeffs, grad_sh, state_sh, lr_sh)
        
    return voxel_grid
```

### Edge Cases and Pitfalls
- **Ill-conditioning**: The high-dimensional, non-convex objective makes direct optimization unstable [S1]. RMSProp is required to stabilize convergence without Hessian computation [S1].
- **TV Weight Selection**: Weights must be fixed per scene type and differ between SH coefficients and opacity [S1]. Mismatched weights cause over-smoothing or leave artifacts [S1].
- **Stochastic Sampling Variance**: Sampling rays and voxels independently per step introduces gradient noise that requires stable optimizer tuning [S1].
- **Boundary Handling**: Implicit neighbor indexing at grid boundaries must clamp or ignore out-of-bounds accesses to prevent indexing errors [G].

## Subtopics

### [Sparse Gradient Optimization](1459-sparse-gradient-optimization.md)
Leverages the observation that early training gradients are highly sparse (fewer than 10% of voxels have non-zero gradients) to restrict updates to active voxels. It distinguishes itself from dense gradient methods by skipping inactive regions, yielding significant speedups.

## Operating characteristics
- **Time Complexity**: Per iteration scales linearly with the number of sampled rays and voxels [S1]. Stochastic sampling reduces overhead compared to full-grid passes [S1].
- **Memory Complexity**: Proportional to the sparse voxel grid size and the number of active voxels [S1].
- **Performance**: TV regularization effectively removes visual artifacts in forward-facing scenes, though PSNR metrics do not always correlate with the qualitative improvement [S1].
- **Strengths**: Suppresses noise and artifacts; handles ill-conditioning via RMSProp; extends to unbounded and 360° scenes [S1].
- **Limitations**: Requires careful per-scene TV weight tuning; stochastic sampling introduces gradient variance; not optimized for real-time inference without further acceleration [S1].
- **When to Use**: When visual quality and artifact suppression are prioritized over raw PSNR, and when working with sparse voxel grids prone to high-frequency noise [S1].
- **When Not to Use**: When computational resources are extremely limited and stochastic sampling overhead is prohibitive, or when memory bandwidth constraints prevent storing per-voxel SH coefficients [S1].

## Background
Plenoxel Optimization with TV Regularization was introduced in the "Plenoxels: Radiance Fields without Neural Networks" paper [S1]. It addresses the overfitting and artifact issues inherent in explicit voxel radiance field optimization by adding gradient-based penalties [S1]. The method adopts the learning rate schedules from JAXNeRF and Mip-NeRF while replacing second-order solvers with RMSProp for practical scalability [S1]. The regularization strategy draws from established image processing techniques that use total variation to preserve edges while smoothing noise [S1].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
[S2] adler-j/learned_gradient_tomography, https://github.com/adler-j/learned_gradient_tomography, fetched 2026-10-27 (unofficial)
