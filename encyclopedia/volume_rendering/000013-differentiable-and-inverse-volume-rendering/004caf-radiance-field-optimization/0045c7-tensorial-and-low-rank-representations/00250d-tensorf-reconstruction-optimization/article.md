# TensoRF Reconstruction Optimization
Also called: Tensorial Radiance Field Optimization

## Overview
TensoRF Reconstruction Optimization is the iterative gradient-based procedure used to fit a tensorial radiance field to a set of multi-view images with known camera poses [S1, unofficial]. It minimizes the L2 rendering loss between synthesized and ground-truth pixel colors, distinguishing itself from standard implicit field optimization by enforcing structural regularization (L1 and Total Variation) on low-rank tensor factors and employing a dynamic coarse-to-fine upsampling strategy to mitigate overfitting and local minima [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $M$ multi-view images $\{I_m\}_{m=1}^M$ with known camera poses and intrinsics.
*   **Output:** Optimized tensor factors (global mode vectors and matrices) representing the density $\sigma$ and color $\mathbf{c}$ fields.

**Data Structures:**
The radiance field is represented using Vector-Matrix (VM) or Canonical Polyadic (CP) decomposition. For a component $r$, the field is approximated by summing rank-1 components. The density $\sigma$ and appearance features $\mathbf{c}$ are computed by interpolating mode vectors and matrices. Specifically, for a component $r$, the density contribution involves terms like $\mathbf{v}_X(x) \cdot \mathbf{M}_{YZ}(y,z)$, where $\mathbf{v}$ is a vector and $\mathbf{M}$ is a matrix [3026-tensorf.md]. The implementation uses a feature grid with $N^3$ voxels, where the resolution per dimension is derived from the bounding box shape [S1, unofficial].

**Parameters:**
*   **Initial Resolution:** $N_0^3$ voxels, with $N_0 = 128$ [S1, unofficial].
*   **Final Resolution:** $N^3$ voxels [S1, unofficial].
*   **Upsampling Steps:** 2000, 3000, 4000, 5500, 7000 [S1, unofficial].
*   **Interpolation:** Linear in logarithmic space between $N_0^3$ and $N^3$ [S1, unofficial].
*   **Optimizer:** Adam [S1, unofficial].
*   **Batch Size:** 4096 pixel rays [S1, unofficial].
*   **Feature Decoding:** Small MLP (2 FC layers, 128 hidden, ReLU) or 3rd-order Spherical Harmonics (SH) with 27 features [S1, unofficial].

**Regularization:**
To prevent overfitting and local minima, regularization is applied to the tensor factors:
1.  **L1 Norm Loss:** Encourages sparsity in the factors. Adequate for most datasets [S1, unofficial].
2.  **Total Variation (TV) Loss:** Promotes smoothness. Used for real datasets with few input images (e.g., LLFF) or imperfect capture conditions (e.g., Tanks and Temples with varying exposure/inconsistent masks) [S1, unofficial].

**Algorithm:**
1.  Initialize tensor factors (global vectors and matrices) at resolution $N_0^3$ [S1, unofficial].
2.  For each optimization step $t$:
    a.  If $t$ is in $\{2000, 3000, 4000, 5500, 7000\}$, upsample the XYZ-mode vector and matrix factors linearly and bilinearly to the current target resolution [S1, unofficial].
    b.  Sample rays and compute rendered pixel colors using the current tensor factors and feature decoding function [S1, unofficial].
    c.  Compute L2 rendering loss: $\mathcal{L}_{render} = \sum \| \mathbf{C}_{rendered} - \mathbf{C}_{GT} \|^2$, where $\mathbf{C}_{rendered}$ are the rendered colors and $\mathbf{C}_{GT}$ are ground truth colors [S1, unofficial].
    d.  Compute regularization loss $\mathcal{L}_{reg}$ based on L1 norm and/or TV loss on the factors [S1, unofficial].
    e.  Compute total loss $\mathcal{L} = \mathcal{L}_{render} + \lambda \mathcal{L}_{reg}$ [S1, unofficial].
    f.  Update tensor factors using Adam optimizer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn as nn
import torch.optim as optim

def compute_tv_loss(factors):
    """Compute Total Variation loss on tensor factors."""
    loss = 0.0
    for factor in factors:
        # Compute differences along each dimension
        diff_x = factor[:, 1:, :] - factor[:, :-1, :]
        diff_y = factor[:, :, 1:] - factor[:, :, :-1]
        diff_z = factor[1:, :, :] - factor[:-1, :, :]
        loss += torch.mean(torch.abs(diff_x)) + \
                torch.mean(torch.abs(diff_y)) + \
                torch.mean(torch.abs(diff_z))
    return loss

def render_ray(ray_origins, ray_directions, factors, resolution):
    """
    Render a batch of rays using tensor factors.
    Simplified for demonstration; actual implementation involves
    trilinear interpolation and volume rendering integration.
    """
    # Sample points along rays
    # Interpolate factors at sample points using trilinear interpolation
    # Compute density sigma and color c
    # Perform volume rendering (alpha compositing)
    # Return rendered colors
    pass

def optimize_tensorf(images, poses, N0=128, N_final=256, steps=[2000, 3000, 4000, 5500, 7000], 
                     use_tv_loss=False, l1_lambda=1e-4, tv_lambda=1e-4):
    """
    Optimize TensoRF factors.
    """
    # Initialize factors at N0^3
    # Vectors and Matrices initialization [G]
    factors = initialize_factors(N0)
    
    optimizer = optim.Adam(factors, lr=0.001) # [G] Typical LR
    
    total_steps = 8000 # [G] Example total steps
    
    for t in range(total_steps):
        # Coarse-to-fine upsampling
        if t in steps:
            # Calculate target resolution linearly in log space
            # Upsample vectors (linear) and matrices (bilinear)
            factors = upsample_factors(factors, t, N0, N_final, steps)
        
        # Sample rays
        ray_origins, ray_directions = sample_rays(images, poses, batch_size=4096)
        
        # Render
        rendered_colors = render_ray(ray_origins, ray_directions, factors, get_current_resolution(factors))
        
        # Get ground truth colors
        gt_colors = get_gt_colors(ray_origins, ray_directions, images)
        
        # Compute L2 loss
        render_loss = torch.mean((rendered_colors - gt_colors) ** 2)
        
        # Compute regularization
        l1_loss = sum(torch.norm(f, p=1) for f in factors)
        reg_loss = l1_lambda * l1_loss
        
        if use_tv_loss:
            tv_loss = compute_tv_loss(factors)
            reg_loss += tv_lambda * tv_loss
            
        loss = render_loss + reg_loss
        
        # Backpropagate
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
    return factors
```

## Subtopics

### [TensoRF Training Loop](8644-tensorf-training-loop.md)
The iterative optimization procedure used to minimize the L2 rendering loss of a tensorial radiance field by adjusting its low-rank tensor parameters. It distinguishes itself from standard NeRF training by incorporating specific regularizations (orthogonal, L1 density, and Total Variation) tailored to the decomposed tensor structure, alongside dynamic voxel resolution updates via an alpha mask.

## Operating characteristics
*   **Time Complexity:** Depends on the number of optimization steps and the resolution. The coarse-to-fine strategy reduces initial computation cost.
*   **Memory Complexity:** Scales with the number of tensor factors, which is significantly lower than dense voxel grids or large MLPs for equivalent resolution.
*   **Performance:** Evaluated on a single Tesla V100 GPU (16GB) with a batch size of 4096 pixel rays [S1, unofficial].
*   **Strengths:** Effective regularization prevents overfitting and local minima. Coarse-to-fine upsampling improves quality.
*   **Limitations:** Requires careful tuning of regularization parameters and upsampling steps. May struggle with very few input images without TV loss.
*   **When to use:** Suitable for multi-view reconstruction where memory efficiency and structural regularization are beneficial.
*   **When not to use:** Not ideal for real-time inference without further optimization (e.g., pruning) or for scenes with extreme view-dependent effects not captured by the low-rank assumption.

## Background
TensoRF was introduced in the paper "TensoRF: Tensorial Radiance Fields" [S1, unofficial]. The method leverages tensor factorization to represent radiance fields, building on the principles of low-rank representations. It addresses issues of overfitting and local minima common in gradient-based optimization of implicit fields by introducing specific regularization terms and a dynamic resolution update strategy.

## Sources
[S1] PDF TensoRF: Tensorial Radiance Fields - ECVA, https://www.ecva.net/papers/eccv_2022/papers_ECCV/papers/136920332.pdf, fetched 2026-10-29 (unofficial)
