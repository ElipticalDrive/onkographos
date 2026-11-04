# Plenoxel Optimization with TV Regularization

Also called: Chambolle-Pock TV, Plenoxels TV and sparsity regularization, TV regularization, Total Variation Reconstruction

## Overview
Plenoxel Optimization with TV Regularization trains sparse voxel-based radiance fields by combining mean squared error (MSE) reconstruction loss with Total Variation (TV) regularization on voxel opacities ($\sigma$) and spherical harmonic (SH) coefficients [S1]. It addresses visual artifacts and ill-conditioning in high-dimensional, non-convex optimization by explicitly penalizing high-frequency spatial variations in the voxel grid [S1]. It builds on [Plenoxels](52-plenoxels.md) and [Plenoxels regularization ablation](901-plenoxels-regularization-ablation.md).

## How it works
**Inputs:**
- Ground truth images and corresponding camera poses [S1].
- A sparse voxel grid structure containing opacity ($\sigma$) and SH coefficients for each active voxel [S1].
- Optimization hyperparameters: $\lambda_{TV}$ (TV weight), learning rates for $\sigma$ and SH coefficients, and batch sizes for rays $\mathcal{R}$ and voxels $\mathcal{V}$ [S1].

**Outputs:**
- Optimized voxel opacities and SH coefficients [S1].

**Data Structures:**
- **Voxel Grid:** A sparse 3D grid where each active voxel stores a scalar opacity $\sigma$ and a vector of SH coefficients [S1].
- **Ray Sample $\mathcal{R}$:** A stochastic subset of training rays used for the reconstruction loss [S1].
- **Voxel Sample $\mathcal{V}$:** A stochastic subset of active voxels used for the TV loss [S1].

**Formulas:**
The total loss is defined as [S1]:
$$
\mathcal{L} = \mathcal{L}_{recon} + \lambda_{TV} \mathcal{L}_{TV}
$$

The reconstruction loss is the MSE between rendered color $\hat{C}(\mathbf{r})$ and ground truth $C(\mathbf{r})$ [S1]:
$$
\mathcal{L}_{recon} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \|C(\mathbf{r}) - \hat{C}(\mathbf{r})\|_2^2
$$

The TV regularization loss is the sum of isotropic gradients over sampled voxels and SH dimensions [S1]:
$$
\mathcal{L}_{TV} = \frac{1}{|\mathcal{V}|} \sum_{\substack{\mathbf{v} \in \mathcal{V} \\ d \in [D]}} \sqrt{\Delta_x^2(\mathbf{v}, d) + \Delta_y^2(\mathbf{v}, d) + \Delta_z^2(\mathbf{v}, d)}
$$
where $D$ is the dimensionality of the SH coefficients, and $\Delta^2_x(\mathbf{v}, d)$ is the squared difference between the $d$-th value in voxel $\mathbf{v}=(i,j,k)$ and its neighbor $(i+1,j,k)$, normalized by the voxel resolution [S1]. Analogous definitions apply for $\Delta^2_y$ and $\Delta^2_z$ [S1].

**Parameters:**
- $\lambda_{TV}$: Fixed per scene type (bounded, forward-facing, 360°). Different weights are used for SH coefficients and $\sigma$ [S1].
- Optimizer: RMSProp, chosen to handle ill-conditioning without the cost of second-order methods [S1].
- Learning Rate: Fixed for all scenes in main experiments, but tuned separately for opacity and harmonic coefficients [S1].

**Algorithm Steps:**
1. **Sample Rays and Voxels:** For each optimization step, draw a stochastic batch of rays $\mathcal{R}$ for reconstruction and a stochastic batch of voxels $\mathcal{V}$ for TV regularization [S1].
2. **Render:** Compute rendered colors $\hat{C}(\mathbf{r})$ for all $\mathbf{r} \in \mathcal{R}$ using differentiable volume rendering [S1].
3. **Compute Reconstruction Loss:** Calculate $\mathcal{L}_{recon}$ using the formula above [S1].
4. **Compute TV Loss:** For each voxel $\mathbf{v} \in \mathcal{V}$ and each SH dimension $d \in [D]$, compute the finite differences in x, y, and z directions. Normalize by resolution, square them, sum them, and take the square root to get the isotropic gradient magnitude. Sum over all sampled voxels and dimensions to get $\mathcal{L}_{TV}$ [S1].
5. **Combine Losses:** Compute $\mathcal{L} = \mathcal{L}_{recon} + \lambda_{TV} \mathcal{L}_{TV}$ [S1].
6. **Backpropagate:** Compute gradients of $\mathcal{L}$ with respect to voxel opacities and SH coefficients [S1].
7. **Update:** Update parameters using RMSProp [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def optimize_plenoxels_tv(voxel_grid, gt_images, cameras, lambda_tv, lr_sigma, lr_sh, n_iters, batch_size_rays, batch_size_voxels):
    # Initialize RMSProp optimizers for opacity and SH coefficients [G]
    opt_sigma = RMSProp(voxel_grid.opacities, lr=lr_sigma)
    opt_sh = RMSProp(voxel_grid.sh_coeffs, lr=lr_sh)

    for step in range(n_iters):
        # 1. Stochastic sampling of rays and voxels [S1]
        rays = sample_rays(cameras, batch_size_rays)
        voxels = sample_voxels(voxel_grid, batch_size_voxels)

        # 2. Differentiable volume rendering [S1]
        rendered_colors = render_rays(voxel_grid, rays)
        gt_colors = get_gt_colors(rays, gt_images)

        # 3. Reconstruction loss [S1]
        l_recon = mean_squared_error(rendered_colors, gt_colors)

        # 4. TV regularization loss [S1]
        l_tv_sigma = compute_tv_loss(voxel_grid, voxels, voxel_grid.opacities, is_scalar=True)
        l_tv_sh = compute_tv_loss(voxel_grid, voxels, voxel_grid.sh_coeffs, is_scalar=False)
        l_tv = l_tv_sigma + l_tv_sh

        # 5. Combine losses [S1]
        loss = l_recon + lambda_tv * l_tv

        # 6. Backpropagation and parameter update [S1]
        loss.backward()
        opt_sigma.step()
        opt_sh.step()
        opt_sigma.zero_grad()
        opt_sh.zero_grad()

def compute_tv_loss(voxel_grid, voxels, field, is_scalar):
    """Computes the isotropic TV loss per voxel and dimension as defined in [S1]."""
    tv_sum = 0.0
    count = 0
    res = voxel_grid.resolution
    for v in voxels:
        if is_scalar:
            val_v = field[v]
            grad_sq_sum = 0.0
            for direction in ['x', 'y', 'z']:
                neighbor = voxel_grid.get_neighbor(v, direction)
                if neighbor is not None:
                    diff = (val_v - field[neighbor]) / res
                    grad_sq_sum += diff ** 2
            tv_sum += math.sqrt(grad_sq_sum)
        else:
            for d in range(field.shape[1]):
                val_v = field[v, d]
                grad_sq_sum = 0.0
                for direction in ['x', 'y', 'z']:
                    neighbor = voxel_grid.get_neighbor(v, direction)
                    if neighbor is not None:
                        diff = (val_v - field[neighbor, d]) / res
                        grad_sq_sum += diff ** 2
                tv_sum += math.sqrt(grad_sq_sum)
        count += 1
    return tv_sum / count if count > 0 else 0.0

# Standard helper definitions for self-contained pseudocode [G]
def sample_rays(cameras, n): return [cameras[i] for i in range(n)]
def sample_voxels(grid, n): return list(grid.active_voxels)[:n]
def render_rays(grid, rays): return [grid.render_ray(r) for r in rays]
def get_gt_colors(rays, images): return [images[r.image_id] for r in rays]
def mean_squared_error(pred, target): return sum((p - t)**2 for p, t in zip(pred, target)) / len(pred)
```

## Subtopics

### Sparse Gradient Optimization
This variant leverages the observation that during the first 1-2 minutes of optimization (or first 12,800 batches for synthetic scenes), fewer than 10% of voxels have non-zero gradients, with as few as 1% updating per batch in synthetic cases [S1]. This sparsity allows for significant computational savings by only updating active voxels [S1]. Implementation requires a custom PyTorch CUDA extension for fast differentiable volume rendering to exploit this sparsity efficiently [S1].

## Operating characteristics
- **Time Complexity:** The method uses stochastic sampling for both rays and voxels, making each iteration faster than full-batch methods [S1]. The use of RMSProp avoids the high cost of second-order methods [S1].
- **Performance:** Achieves high-quality results after less than 1.5 minutes (first epoch) on synthetic scenes [S1]. Dramatic speedup in training compared to prior methods [S1].
- **Strengths:** Prevents visual artifacts, handles ill-conditioning, and enables fast training [S1].
- **Limitations:** Requires careful tuning of $\lambda_{TV}$ and learning rates for different parameters [S1]. The non-convex nature of the rendering formula makes optimization challenging [S1].
- **When to use:** When training sparse voxel radiance fields and needing to prevent artifacts and control memory [S1].

## Background
The method originates from the Plenoxels paper, which introduced radiance fields without neural networks using sparse voxel grids and SH coefficients [S1]. TV regularization is applied to the loss function to mitigate visual artifacts observed in ablation studies [S1]. Implementation details for the optimization framework are documented in an unofficial repository [S2, unofficial].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
[S2] adler-j/learned_gradient_tomography, https://github.com/adler-j/learned_gradient_tomography, fetched 2026-10-27 (unofficial)
