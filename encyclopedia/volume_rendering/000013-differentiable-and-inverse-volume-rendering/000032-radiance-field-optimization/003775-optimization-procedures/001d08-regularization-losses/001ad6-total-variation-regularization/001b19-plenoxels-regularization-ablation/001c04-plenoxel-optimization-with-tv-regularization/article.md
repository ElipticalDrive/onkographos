# Plenoxel Optimization with TV Regularization

Also called: Chambolle-Pock TV, Plenoxels TV and sparsity regularization, TV regularization, Total Variation Reconstruction

## Overview
Plenoxel Optimization with TV Regularization is a method for reconstructing radiance fields from sparse voxel grids by minimizing a composite loss function that combines Mean Squared Error (MSE) reconstruction error with isotropic Total Variation (TV) regularization [S1]. It solves the problem of visual artifacts and ill-conditioning inherent in high-dimensional, non-convex radiance field optimization, where standard gradient descent often fails to converge smoothly or produces noisy, artifact-ridden results [S1]. Unlike methods that rely on neural network smoothing or dense grid constraints, this approach explicitly penalizes the spatial gradient of voxel opacities ($\sigma$) and spherical harmonic (SH) coefficients, enforcing piecewise smoothness while preserving sharp boundaries [S1]. It builds on [Plenoxels regularization ablation](901-plenoxels-regularization-ablation.md) by detailing the specific optimization mechanics, stochastic sampling strategies, and solver configurations required to implement this regularization effectively.

## How it works
The method optimizes voxel opacities and SH coefficients using RMSProp to handle ill-conditioning without the computational cost of second-order methods [S1]. The optimization is driven by a total loss function comprising a reconstruction term and a TV regularization term [S1].

### Inputs and Outputs
- **Inputs**:
  - Ground truth image colors $C(\mathbf{r})$ for a set of rays $\mathcal{R}$ [S1].
  - Initial voxel grid containing opacity $\sigma$ and SH coefficients [S1].
  - Scene type (bounded, forward-facing, or 360°) to determine TV weights [S1].
- **Outputs**:
  - Optimized voxel opacities and SH coefficients [S1].

### Data Structures
The core data structure is a sparse voxel grid. Each active voxel $\mathbf{v} = (i, j, k)$ stores:
- Opacity $\sigma_{\mathbf{v}}$ (scalar) [S1].
- SH coefficients $\mathbf{c}_{\mathbf{v}} \in \mathbb{R}^D$, where $D$ is the dimensionality of the SH basis [S1].
- Neighbor pointers or implicit indexing for gradient computation [S1].

### Formulas
The total loss $\mathcal{L}$ is defined as [S1]:
$$
\mathcal{L} = \mathcal{L}_{recon} + \lambda_{TV} \mathcal{L}_{TV}
$$

The reconstruction loss $\mathcal{L}_{recon}$ is the MSE between rendered colors $\hat{C}(\mathbf{r})$ and ground truth colors $C(\mathbf{r})$ [S1]:
$$
\mathcal{L}_{recon} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \|C(\mathbf{r}) - \hat{C}(\mathbf{r})\|_2^2
$$

The TV regularization loss $\mathcal{L}_{TV}$ is the isotropic total variation over a stochastic batch of active voxels $\mathcal{V}$ and SH dimensions $d \in [D]$ [S1]:
$$
\mathcal{L}_{TV} = \frac{1}{|\mathcal{V}|} \sum_{\substack{\mathbf{v} \in \mathcal{V} \\ d \in [D]}} \sqrt{\Delta_x^2(\mathbf{v}, d) + \Delta_y^2(\mathbf{v}, d) + \Delta_z^2(\mathbf{v}, d)}
$$

Here, $\Delta_x^2(\mathbf{v}, d)$ is the squared difference between the $d$-th value in voxel $\mathbf{v}=(i,j,k)$ and its neighbor $(i+1,j,k)$, normalized by the voxel resolution [S1]. Analogous definitions apply for $\Delta_y^2$ and $\Delta_z^2$ [S1].

### Parameters
- **TV Weight ($\lambda_{TV}$)**: Fixed per scene type (bounded, forward-facing, 360°) and differs for SH coefficients and opacity $\sigma$ [S1]. [S2, unofficial] suggests a weight of 0.3 for the L1 norm in a similar TV formulation.
- **Optimizer**: RMSProp [S1].
- **Learning Rate**: Fixed for all scenes in main experiments, but tuned separately for opacity and harmonic coefficients [S1].
- **Stochastic Sampling**: Rays $\mathcal{R}$ and voxels $\mathcal{V}$ are sampled stochastically per optimization step [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_plenoxels_tv(voxel_grid, rays, gt_colors, scene_type, n_iters):
    """
    Optimizes Plenoxel grid using TV regularization and RMSProp.
    
    Args:
        voxel_grid: Sparse voxel grid with opacity and SH coefficients.
        rays: List of ray origins and directions.
        gt_colors: Ground truth colors for rays.
        scene_type: 'bounded', 'forward-facing', or '360'.
        n_iters: Number of optimization iterations.
        
    Returns:
        Optimized voxel_grid.
    """
    # Initialize optimizer state for opacity and SH coefficients
    # Note: Learning rates are tuned separately for sigma and SH [S1]
    optimizer_sigma = RMSProp(voxel_grid.opacities, lr=lr_sigma)
    optimizer_sh = RMSProp(voxel_grid.sh_coeffs, lr=lr_sh)
    
    # Get TV weights based on scene type [S1]
    lambda_tv_sigma = get_tv_weight(scene_type, 'sigma')
    lambda_tv_sh = get_tv_weight(scene_type, 'sh')
    
    for step in range(n_iters):
        # 1. Stochastic Sampling [S1]
        # Sample a subset of rays for reconstruction loss
        batch_rays = sample_rays(rays, size=batch_size_rays)
        # Sample a subset of active voxels for TV loss
        batch_voxels = sample_voxels(voxel_grid, size=batch_size_voxels)
        
        # 2. Compute Reconstruction Loss [S1]
        rendered_colors = render_rays(voxel_grid, batch_rays)
        l_recon = mean_squared_error(gt_colors[batch_rays], rendered_colors)
        
        # 3. Compute TV Regularization Loss [S1]
        l_tv_sigma = compute_isotropic_tv(voxel_grid.opacities, batch_voxels)
        l_tv_sh = compute_isotropic_tv(voxel_grid.sh_coeffs, batch_voxels)
        
        # 4. Combine Losses [S1]
        # Note: Different weights for SH and sigma [S1]
        l_tv = lambda_tv_sigma * l_tv_sigma + lambda_tv_sh * l_tv_sh
        total_loss = l_recon + l_tv
        
        # 5. Backpropagation and Update [S1]
        # Compute gradients of total_loss w.r.t. opacities and SH coefficients
        grad_sigma = compute_gradient(total_loss, voxel_grid.opacities)
        grad_sh = compute_gradient(total_loss, voxel_grid.sh_coeffs)
        
        # Update using RMSProp [S1]
        optimizer_sigma.step(grad_sigma)
        optimizer_sh.step(grad_sh)
        
    return voxel_grid

def compute_isotropic_tv(values, batch_voxels):
    """
    Computes isotropic TV for a given field (sigma or SH coefficients).
    """
    tv_sum = 0.0
    count = 0
    for v in batch_voxels:
        for d in range(values.dim): # D is SH dimensionality
            # Get value and neighbors normalized by resolution [S1]
            val = values[v, d]
            val_x = values[neighbor(v, 'x'), d]
            val_y = values[neighbor(v, 'y'), d]
            val_z = values[neighbor(v, 'z'), d]
            
            # Squared differences normalized by resolution [S1]
            dx = (val - val_x) / voxel_resolution
            dy = (val - val_y) / voxel_resolution
            dz = (val - val_z) / voxel_resolution
            
            tv_sum += sqrt(dx**2 + dy**2 + dz**2)
            count += 1
            
    return tv_sum / count if count > 0 else 0.0
```

### Edge Cases and Pitfalls
- **Ill-conditioning**: Direct optimization is challenging due to high dimensionality and non-convexity [S1]. RMSProp is used to mitigate this, avoiding the need for Hessian computation [S1].
- **TV Weight Selection**: Weights must be fixed per scene type and differ for SH vs. opacity [S1]. Incorrect weights can lead to over-smoothing or insufficient artifact removal [S1].
- **Stochastic Sampling Bias**: Using different stochastic samples for rays and voxels requires careful balancing to ensure stable gradients [S1].
- **Order of Functionals**: If using a solver like PDHG (as in [S2, unofficial]), the order of functionals in `SeparableSum` must correspond to the order of operators in `BroadcastOperator` [S2, unofficial].

## Subtopics

### [Sparse Gradient Optimization](1459-sparse-gradient-optimization.md)
Leverages the sparsity of gradients in early training stages to focus computation on active voxels, offering significant speedups over dense gradient methods.

## Operating characteristics
- **Time Complexity**: Per iteration, complexity is proportional to the number of sampled rays and voxels [S1]. Stochastic sampling reduces this compared to full-grid updates [S1].
- **Memory Complexity**: Depends on the sparse voxel grid size and the number of active voxels [S1].
- **Performance**: TV regularization reduces visual artifacts, particularly in forward-facing scenes, although PSNR improvements may not always reflect this qualitative gain [S1].
- **Strengths**: Effective at removing noise and artifacts; handles ill-conditioning well with RMSProp; extends to unbounded scenes [S1].
- **Limitations**: Requires careful tuning of TV weights per scene type; stochastic sampling may introduce variance; not suitable for real-time inference without further optimization [S1].
- **When to Use**: When visual quality is prioritized over raw PSNR, and when dealing with sparse voxel grids prone to artifacts [S1].
- **When Not to Use**: When computational resources are extremely limited and stochastic sampling overhead is prohibitive, or when neural network-based smoothing is preferred [S1].

## Background
Plenoxels optimization with TV regularization was introduced in the "Plenoxels: Radiance Fields without Neural Networks" paper [S1]. It addresses the limitations of neural radiance fields by using explicit voxel representations, which require regularization to prevent overfitting and artifacts [S1]. The method builds on earlier work in radiance field optimization, such as JAXNeRF and Mip-NeRF, adopting their learning rate schedules [S1]. The use of TV regularization in this context is inspired by general image processing techniques [S1].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
[S2] adler-j/learned_gradient_tomography, https://github.com/adler-j/learned_gradient_tomography, fetched 2026-10-27 (unofficial)
