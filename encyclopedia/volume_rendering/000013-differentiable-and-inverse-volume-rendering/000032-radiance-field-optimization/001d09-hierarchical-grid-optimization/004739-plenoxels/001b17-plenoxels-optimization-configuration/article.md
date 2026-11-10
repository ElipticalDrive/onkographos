# Plenoxels Optimization Configuration

Also called: Plenoxels optimizer settings, RMSProp vs SGD for Plenoxels

## Overview
Plenoxels Optimization Configuration defines the hyperparameter settings and training protocols required to optimize the voxel opacity ($\sigma$) and spherical harmonic (SH) coefficients of a sparse 3D grid representation. It solves the problem of efficiently converging to high-fidelity radiance fields without neural networks by selecting appropriate gradient-based optimizers and learning rate schedules. It is distinguished from sibling methods like [Explicit Radiance Fields](806-explicit-radiance-fields.md) and [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-optimization-with-pruning.md) by its specific reliance on RMSProp for SH coefficients and its robustness to learning rate schedule variations, while building on the differentiable volume rendering principles of [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md).

## How it works
The optimization process updates the scene parameters (voxel opacity and SH coefficients) to minimize the difference between rendered images and ground truth multi-view images.

**Inputs and Outputs:**
- **Inputs:** Multi-view images, camera poses, initial sparse voxel grid (initialized via unprojection or space carving), and hyperparameters (optimizer type, learning rates, schedule).
- **Outputs:** Optimized voxel opacity field $\sigma$ and spherical harmonic coefficient tensors $\mathbf{c}_{SH}$.

**Data Structures:**
The scene is represented as a sparse 3D grid. Each active voxel stores:
1.  **Opacity ($\sigma$):** A scalar value representing density.
2.  **Spherical Harmonic Coefficients ($\mathbf{c}_{SH}$):** A vector of coefficients for each RGB channel. For degree $L$, there are $(L+1)^2$ coefficients per channel.

**Numbered Steps:**
1.  **Initialization:** Initialize the voxel grid. Opacity $\sigma$ can be initialized to small positive values or derived from space carving. SH coefficients are typically initialized to zero or based on the average color of the voxel's view.
2.  **Forward Rendering:** For each training step, sample rays from the input views. Compute the transmittance $T_i$ and accumulated color $\hat{C}(\mathbf{r})$ using the standard emission-absorption model [S1][S5]:
    $$
    \hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
    $$
    where $\mathbf{c}_i$ is the color computed by evaluating the SH functions at the viewing direction for voxel $i$.
3.  **Loss Computation:** Compute the photometric loss (e.g., $L_1$ or $L_2$) between $\hat{C}(\mathbf{r})$ and the ground truth image color. Regularization terms such as Total Variation (TV) loss on $\sigma$ and SH coefficients may be added [S1].
4.  **Gradient Computation:** Compute gradients $\nabla_\sigma \mathcal{L}$ and $\nabla_{\mathbf{c}_{SH}} \mathcal{L}$ with respect to the voxel parameters. This can be done via reverse-mode automatic differentiation or explicit gradient formulas [S5].
5.  **Parameter Update:** Update $\sigma$ and $\mathbf{c}_{SH}$ using the configured optimizer and learning rate schedule.

**Parameters and Typical Values:**
-   **Optimizer:** RMSProp is preferred over SGD, particularly for optimizing SH coefficients [S1].
-   **Learning Rate Schedule:**
    -   **Best Practice:** Exponential decay for SH coefficients and a delayed schedule for $\sigma$ [S1].
    -   **Alternative:** Constant learning rate for both [S1].
-   **Resolution:** $256^3$ is used in baseline evaluations [S1].
-   **SH Degree:** Degree 9 is used in some implementations (e.g., space carving variant) [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def optimize_plenoxels(voxel_grid, images, poses, optimizer_type='RMSProp', 
                       lr_schedule_sh='exp', lr_schedule_sigma='delayed', 
                       num_steps=10000):
    """
    Optimizes Plenoxels voxel grid parameters.
    
    Args:
        voxel_grid: Sparse 3D grid with attributes sigma (opacity) and sh_coeffs (SH coefficients).
        images: List of ground truth images.
        poses: List of camera poses.
        optimizer_type: 'RMSProp' or 'SGD'.
        lr_schedule_sh: 'exp' or 'constant'.
        lr_schedule_sigma: 'delayed' or 'constant'.
        num_steps: Number of optimization steps.
        
    Returns:
        Optimized voxel_grid.
    """
    # Initialize optimizer state
    if optimizer_type == 'RMSProp':
        optimizer_sigma = RMSProp(voxel_grid.sigma, lr=1e-3)
        optimizer_sh = RMSProp(voxel_grid.sh_coeffs, lr=1e-3)
    elif optimizer_type == 'SGD':
        optimizer_sigma = SGD(voxel_grid.sigma, lr=1e-3)
        optimizer_sh = SGD(voxel_grid.sh_coeffs, lr=1e-3)
        
    for step in range(num_steps):
        # Sample a random view
        idx = random.randint(0, len(images) - 1)
        gt_image = images[idx]
        pose = poses[idx]
        
        # Forward pass: render image from current voxel grid
        rendered_image = render_ray_marching(voxel_grid, pose)
        
        # Compute loss
        loss = l1_loss(rendered_image, gt_image)
        
        # Add regularization if configured (e.g., TV loss)
        # loss += tv_loss(voxel_grid.sigma) + tv_loss(voxel_grid.sh_coeffs)
        
        # Backward pass
        loss.backward()
        
        # Update parameters based on schedule
        if lr_schedule_sh == 'exp':
            current_lr_sh = initial_lr_sh * (decay_factor ** step)
        else:
            current_lr_sh = initial_lr_sh
            
        if lr_schedule_sigma == 'delayed':
            # Delayed schedule: start updating sigma after some steps or with different decay
            if step < warmup_steps:
                current_lr_sigma = 0
            else:
                current_lr_sigma = initial_lr_sigma * (decay_factor ** (step - warmup_steps))
        else:
            current_lr_sigma = initial_lr_sigma
            
        # Apply updates
        optimizer_sigma.step(lr=current_lr_sigma)
        optimizer_sh.step(lr=current_lr_sh)
        
        # Zero gradients
        optimizer_sigma.zero_grad()
        optimizer_sh.zero_grad()
        
    return voxel_grid
```

**Edge Cases and Pitfalls:**
-   **Precision:** SH coefficients can be large; ensure numerical stability in SH evaluation [S1].
-   **Boundaries:** Voxel boundaries must be handled correctly during interpolation to avoid artifacts [S1].
-   **Degenerate Input:** If the initial voxel grid is too sparse, optimization may fail to converge. Space carving can help initialize a denser grid [S2, unofficial].

## Subtopics

### Space Carving and Opaque Voxels Optimization with Spherical Harmonics
A technique composed of two steps: space carving and opaque voxels optimization, used to generate neural radiance fields based on plenoxels [S2, unofficial]. It differs from static grid optimization by using space carving to initialize the voxel grid, which helps in defining the scene geometry more accurately. Spherical harmonics of degree 9 are used to render the shading of each voxel [S2, unofficial]. Running the implementation `main_carve_sph.py` takes approximately 4 minutes on a single GPU [S2, unofficial].

## Operating characteristics
-   **Time Complexity:** Optimization time depends on the number of voxels, samples per ray, and training steps. RMSProp is generally faster to converge than SGD for SH coefficients [S1].
-   **Memory Complexity:** Memory usage is dominated by the storage of voxel opacity and SH coefficients. For a $256^3$ grid with degree 9 SH, memory usage is significant but manageable on modern GPUs [S1].
-   **Measured Performance:** On 8 synthetic NeRF scenes at $256^3$ resolution, RMSProp for both SH and $\sigma$ achieves PSNR 30.57, SSIM 0.950, LPIPS 0.065. SGD for both drops to PSNR 29.35, SSIM 0.932, LPIPS 0.087 [S1].
-   **Strengths:** Robust to variations in learning rate schedule and optimizer choice, though RMSProp is preferred [S1]. Efficient compared to neural network-based methods.
-   **Limitations:** Performance degrades with complex lighting and high-frequency details if SH degree is low. Memory usage scales with grid resolution and SH degree.
-   **When to Use:** Suitable for static scenes with moderate complexity where high-quality novel view synthesis is required without the overhead of neural networks.
-   **When Not to Use:** Not suitable for dynamic scenes or scenes with very high-frequency details that require higher SH degrees or neural representations.

## Background
Plenoxels Optimization Configuration originates from the Plenoxels paper, which introduced a neural network-free radiance field representation using sparse voxel grids and spherical harmonics [S1]. The method was developed to address the computational and memory inefficiencies of neural radiance fields (NeRFs) while maintaining high rendering quality. The specific optimization configurations, such as the preference for RMSProp and delayed learning rate schedules for opacity, were empirically determined to provide the best balance of convergence speed and rendering quality [S1]. The method has been widely adopted in subsequent research and applications, including variants like [PlenOctree Optimization](1291-plenoctree-optimization.md) and [Sparse Voxel Grid Optimization](2568-sparse-voxel-grid-optimization.md).

## Sources
[S1] Plenoxels: Radiance Fields without Neural Networks, https://doi.org/10.1109/cvpr52688.2022.00542, fetched 2026-10-26
[S2] nissmar/Radiance-Fields, https://github.com/nissmar/Radiance-Fields, fetched 2026-10-27 (unofficial)
