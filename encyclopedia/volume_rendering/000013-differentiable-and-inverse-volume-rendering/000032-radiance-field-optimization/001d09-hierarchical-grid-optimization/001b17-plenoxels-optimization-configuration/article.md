# Plenoxels Optimization Configuration

Also called: Plenoxels optimizer settings, RMSProp vs SGD for Plenoxels

## Overview
Plenoxels Optimization Configuration defines the specific hyperparameter settings and learning rate schedules required to train a static voxel grid with spherical harmonics (SH) coefficients, as introduced in the Plenoxels method [S1]. It solves the problem of efficiently optimizing high-dimensional SH coefficients and voxel opacities ($\sigma$) without the computational overhead of neural network backpropagation [S1]. It is distinguished from [PlenOctree Optimization](1291-plenoctree-optimization.md) by its use of a fixed-resolution dense or sparse voxel grid rather than a hierarchical octree, and from [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-optimization-with-pruning.md) by its reliance on specific optimizer choices (RMSProp) and delayed learning rate schedules rather than geometric pruning strategies [S1].

## How it works
The optimization minimizes the photometric loss $\mathcal{L}$ between rendered colors $\hat{C}(\mathbf{r})$ and ground-truth colors $C(\mathbf{r})$ [S1]. The method optimizes two sets of parameters: voxel opacity $\sigma$ and spherical harmonic coefficients $c_{lm}$ for each voxel [S1].

**Inputs and Outputs:**
*   **Inputs:** Training images, camera poses, initial voxel grid (often initialized via space carving or random initialization), and hyperparameters (optimizer type, learning rates, schedules).
*   **Outputs:** Optimized voxel grid containing $\sigma$ and SH coefficients.

**Data Structures:**
*   **Voxel Grid:** A 3D array of resolution $N \times N \times N$ (e.g., $256^3$ or $128^3$) [S1][S2, unofficial].
*   **Voxel Data:** Each voxel stores:
    *   Opacity $\sigma \in \mathbb{R}$ (or $\mathbb{R}^+$).
    *   SH Coefficients: A vector of coefficients for degree $L$. For degree $L$, there are $(L+1)^2$ coefficients per color channel (R, G, B) [S1][S2, unofficial].

**Optimization Steps:**
1.  **Initialization:** Initialize $\sigma$ and SH coefficients. Common initialization involves setting $\sigma$ to small positive values or using space carving results [S1][S2, unofficial].
2.  **Ray Sampling:** Sample rays from training images [S1].
3.  **Forward Pass:** Render pixels using differentiable volume rendering with the current $\sigma$ and SH coefficients [S1].
4.  **Loss Computation:** Compute MSE loss $\mathcal{L}$ [S1].
5.  **Backpropagation:** Compute gradients for $\sigma$ and SH coefficients [S1].
6.  **Update:** Update parameters using the configured optimizer and learning rate schedule [S1].

**Parameters and Schedules:**
*   **Optimizer:** RMSProp is preferred over SGD, particularly for optimizing SH coefficients [S1].
*   **Learning Rate Schedule:**
    *   **Best Practice:** Delayed learning rate schedule for $\sigma$ and constant learning rate for SH coefficients [S1].
    *   **Alternative:** Constant learning rate for both $\sigma$ and SH [S1].
    *   **Note:** The method is robust to variations in learning rate schedule [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_plenoxels(voxel_grid, training_data, optimizer_type='RMSProp', 
                       lr_sigma_schedule='delayed', lr_sh_schedule='constant',
                       max_steps=10000):
    """
    Optimizes Plenoxels voxel grid.
    
    Args:
        voxel_grid: Object containing sigma and SH coefficients.
        training_data: List of training images and camera poses.
        optimizer_type: 'RMSProp' or 'SGD'.
        lr_sigma_schedule: 'delayed' or 'constant'.
        lr_sh_schedule: 'constant'.
        max_steps: Number of optimization steps.
    """
    # Initialize optimizer
    if optimizer_type == 'RMSProp':
        optimizer = RMSProp(params=[voxel_grid.sigma, voxel_grid.sh_coeffs])
    elif optimizer_type == 'SGD':
        optimizer = SGD(params=[voxel_grid.sigma, voxel_grid.sh_coeffs])
    else:
        raise ValueError("Unsupported optimizer type")

    # Learning rate setup
    base_lr_sigma = 0.001  # Example value, not given in sources
    base_lr_sh = 0.001     # Example value, not given in sources
    
    for step in range(max_steps):
        # 1. Ray Sampling
        rays = sample_rays(training_data)
        
        # 2. Forward Pass
        rendered_colors = render_plenoxels(voxel_grid, rays)
        ground_truth_colors = get_ground_truth(rays)
        
        # 3. Loss Computation
        loss = mse_loss(rendered_colors, ground_truth_colors)
        
        # 4. Backpropagation
        optimizer.zero_grad()
        loss.backward()
        
        # 5. Update with Learning Rate Schedule
        if lr_sigma_schedule == 'delayed':
            # Delayed schedule: e.g., start with 0 LR, then ramp up or use standard decay
            # Note: Exact formula for "delayed" is not specified in sources [G]
            lr_sigma = get_delayed_lr(step, base_lr_sigma)
        else:
            lr_sigma = base_lr_sigma
            
        if lr_sh_schedule == 'constant':
            lr_sh = base_lr_sh
        else:
            lr_sh = get_constant_lr(step, base_lr_sh)
            
        # Apply learning rates
        optimizer.step(lr_sigma=lr_sigma, lr_sh=lr_sh)
        
        # Optional: Regularization (TV loss, Sparsity)
        # loss += lambda_tv * tv_loss(voxel_grid.sigma)
        # loss += lambda_sh * tv_loss(voxel_grid.sh_coeffs)
```

## Subtopics

### Space Carving and Opaque Voxels Optimization with Spherical Harmonics
A technique composed of two steps: space carving and opaque voxels optimization, used to generate neural radiance fields based on plenoxels [S2, unofficial]. It differs from static grid optimization by explicitly using space carving to initialize the voxel grid, which helps in defining the initial geometry [S2, unofficial]. Spherical harmonics of degree 9 are used to render the shading of each voxel [S2, unofficial]. Running the implementation `main_carve_sph.py` takes approximately 4 minutes on a single GPU [S2, unofficial].

### Plenoxels Training
Training implementation of plenoxels that saves the final voxel grid [S2, unofficial]. It differs from the space carving variant by using a standard training loop without explicit space carving initialization [S2, unofficial]. The final voxel grid resolution is 128x128x128 [S2, unofficial]. Running the training implementation `main.py` takes approximately 30 minutes on a single GPU [S2, unofficial].

### [Plenoxels Training with Progressive Growing](1493-plenoxels-training-with-progressive-growing.md)
A training procedure for Plenoxels that uses progressive growing of the receptive field size to optimize a 3D voxel grid from multi-view images, incorporating TV loss and beta loss for regularization. It differs from static grid optimization by starting with a coarse, downsampled representation and refining it, which reduces training time and improves convergence stability.

### [Joint Finetuning for Vector-Quantized Radiance Fields](1871-joint-finetuning-for-vector-quantized-radiance-fields.md)
Joint finetuning is a post-compression optimization procedure applied to vector-quantized radiance fields to recover rendering quality lost during voxel pruning and codebook initialization. It adapts gradient-based optimization to the constraints of vector-quantized structures, building on [Plenoxels Optimization Configuration](929-plenoxels-optimization-configuration.md).

## Operating characteristics
*   **Time Complexity:** Depends on the number of voxels, SH degree, and number of training steps. RMSProp is generally faster to converge than SGD for this task [S1].
*   **Memory Complexity:** Proportional to the voxel grid size and SH degree. For a $256^3$ grid and degree 4 SH, memory usage is significant but manageable on modern GPUs [S1].
*   **Measured Performance:** On 8 synthetic NeRF scenes at $256^3$ resolution, RMSProp for both SH and $\sigma$ achieves PSNR 30.57, SSIM 0.950, LPIPS 0.065 [S1]. SGD for both drops to PSNR 29.35, SSIM 0.932, LPIPS 0.087 [S1].
*   **Strengths:** High quality results with RMSProp; robust to learning rate schedule variations [S1].
*   **Limitations:** SGD performs significantly worse than RMSProp [S1].
*   **When to Use:** When high-quality static radiance field rendering is required and computational resources allow for voxel grid optimization [S1].
*   **When Not to Use:** When real-time rendering on low-end hardware is required without pre-computation, or when dynamic scenes are involved [S1].

## Background
Plenoxels was introduced by Yu et al. in 2022 [S1]. The method leverages the efficiency of voxel grids and spherical harmonics to achieve high-quality novel view synthesis without neural networks [S1]. The optimization configuration was derived from empirical studies on synthetic NeRF scenes [S1].

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
[S2] nissmar/Radiance-Fields, https://github.com/nissmar/Radiance-Fields, fetched 2026-10-27 (unofficial)
