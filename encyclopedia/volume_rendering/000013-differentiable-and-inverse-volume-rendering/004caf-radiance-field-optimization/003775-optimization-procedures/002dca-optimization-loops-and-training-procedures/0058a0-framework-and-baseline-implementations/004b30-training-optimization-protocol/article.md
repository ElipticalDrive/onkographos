# Training & Optimization Protocol

Also called: Splatfacto Training Protocol

## Overview
This protocol defines the training loop, parameter initialization, and optimization hyperparameters for decomposed Gaussian splatting methods, specifically targeting distractor-free rendering. It builds on [Framework and Baseline Implementations](16220-framework-and-baseline-implementations.md) by detailing how static scene Gaussians and distractor points are jointly optimized. It distinguishes itself from standard Gaussian Splatting by explicitly modeling and optimizing distractor elements to improve background reconstruction and reduce artifacts, using specific learning rate schedules and regularization weights [S1].

## How it works
### Inputs and Outputs
**Inputs:**
- Training images and corresponding camera poses.
- Initial static point cloud (typically from COLMAP) [S1].
- Hyperparameters for optimizer, scheduler, and regularization.

**Outputs:**
- Optimized parameters for static Gaussians (means, opacities, scales, rotations, spherical harmonics).
- Optimized parameters for distractor Gaussians (means, opacities, scales, rotations, RGB colors) [S1].

### Data Structures
The method maintains two sets of Gaussian primitives:
1.  **Static Gaussians:** Initialized from COLMAP point cloud. Parameters include mean $\mu$, opacity $\alpha$, scale $s$, rotation $q$ (quaternion), and spherical harmonic coefficients $c_{lm}$ [S1].
2.  **Distractor Gaussians:** Initialized randomly or via specific sampling (1000 points per training image). Parameters include mean $\mu$, opacity $\alpha$, scale $s$, rotation $q$, and RGB color $\mathbf{c}$ [S1].

### Steps
1.  **Initialization:**
    - Initialize static Gaussians using the standard COLMAP point cloud method [S1].
    - Initialize 1000 distractor points per training image [S1].
    - Initialize appearance embeddings if using appearance modeling (e.g., WildGaussians style): per-image embedding $e_j \in \mathbb{R}^{32}$ and per-Gaussian embedding $z_i \in \mathbb{R}^{24}$ with Fourier frequencies [S1].

2.  **Forward Pass:**
    - Render images using the current set of static and distractor Gaussians.
    - Compute photometric loss (e.g., MSE) and perceptual loss (e.g., LPIPS) against ground truth [S2].

3.  **Loss Computation:**
    - The total loss $L$ for a batch of $M$ novel views is:
      $$
      L = \sum_{m=1}^M \left( L_{MSE}(I_{render}^{(m)}, I_{gt}^{(m)}) + \lambda L_{LPIPS}(I_{render}^{(m)}, I_{gt}^{(m)}) \right)
      $$
      where $\lambda = 0.05$ [S2].
    - Apply regularization terms for distractor Gaussians:
      - Alpha regularization: $\lambda_\alpha = 0.002$ [S1].
      - Scale regularization: $\lambda_s = 0$ [S1].
      - Distance regularization: $\lambda_d = 0.01$ [S1].
      - Background regularization: $\lambda_{bg} = 0.15$ [S1].
      - Threshold for regularization $T_\epsilon = 0.003$ [S1].

4.  **Backward Pass and Optimization:**
    - Compute gradients with respect to all parameters.
    - Update parameters using the Adam optimizer [S1].
    - Apply Exponential decay scheduler with decay rates: 0.0001, 0.0002, and 0.00001 [S1].

5.  **Distractor Densification:**
    - Densify distractor Gaussians for the first 15,000 iterations [S1].
    - Stop densification after 15,000 iterations [S1].

6.  **Training Duration:**
    - Train for 30,000 iterations on RobustNeRF, On-the-go, and Mip-NeRF-360 datasets [S1].
    - Train for 200,000 iterations on Photo Tourism datasets [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialization
static_gaussians = initialize_from_colmap()
distractor_gaussians = initialize_distractors(num_points_per_image=1000)

# Optimizer setup
optimizer = Adam(
    params=[static_gaussians.params, distractor_gaussians.params],
    lr=defaults
)
scheduler = ExponentialDecay(decay_rates=[0.0001, 0.0002, 0.00001])

# Hyperparameters
lambda_lpips = 0.05
lambda_alpha_reg = 0.002
lambda_d_reg = 0.01
lambda_bg_reg = 0.15
epsilon_threshold = 0.003
max_densification_iter = 15000
total_iterations = 30000 if dataset in ["RobustNeRF", "On-the-go", "Mip-NeRF-360"] else 200000

for iteration in range(1, total_iterations + 1):
    # Sample training images and cameras
    images, cameras = sample_batch()
    
    # Forward pass
    rendered_images = render(static_gaussians, distractor_gaussians, cameras)
    
    # Loss computation
    loss_mse = compute_mse(rendered_images, images)
    loss_lpips = compute_lpips(rendered_images, images)
    loss_perceptual = lambda_lpips * loss_lpips
    
    # Regularization losses
    reg_alpha = compute_alpha_regularization(distractor_gaussians, lambda_alpha_reg, epsilon_threshold)
    reg_dist = compute_distance_regularization(distractor_gaussians, lambda_d_reg)
    reg_bg = compute_background_regularization(distractor_gaussians, lambda_bg_reg)
    
    total_loss = loss_mse + loss_perceptual + reg_alpha + reg_dist + reg_bg
    
    # Backward pass
    optimizer.zero_grad()
    total_loss.backward()
    
    # Update learning rates for distractor points
    set_lr(distractor_gaussians.quaternions, 0.01)
    set_lr(distractor_gaussians.scales, 0.05)
    set_lr(distractor_gaussians.rgb, 0.025)
    # Means and opacities use Splatfacto defaults [S1]
    
    # Optimization step
    optimizer.step()
    scheduler.step()
    
    # Densification
    if iteration < max_densification_iter:
        densify_distractor_gaussians(distractor_gaussians)
```

### Parameters
- **Optimizer:** Adam [S1].
- **Learning Rates (Distractors):** Quaternions=0.01, Scales=0.05, RGB=0.025 [S1]. Means and opacities use Splatfacto defaults [S1].
- **Scheduler:** Exponential decay with rates 0.0001, 0.0002, 0.00001 [S1].
- **Regularization Weights:** $\lambda_\alpha=0.002$, $\lambda_s=0$, $\lambda_d=0.01$, $\lambda_{bg}=0.15$ [S1].
- **Threshold:** $T_\epsilon=0.003$ [S1].
- **Perceptual Loss Weight:** $\lambda=0.05$ [S2].

### Edge Cases and Pitfalls
- **Distractor Densification:** Must stop after 15,000 iterations to prevent overfitting or instability [S1].
- **Learning Rate Mismatch:** Distractor points require specific learning rates different from static points; using defaults for all may lead to poor convergence [S1].
- **Regularization Threshold:** The threshold $T_\epsilon=0.003$ is critical for controlling the influence of regularization terms [S1].

## Subtopics

### Perceptual Loss Weighting
Uses LPIPS loss with weight $\lambda=0.05$ to improve perceptual quality during optimization [S2].

### Symmetric View Mirroring
A data augmentation technique for single-view reconstruction of roughly symmetric objects (e.g., shoes) by mirroring input images to simulate opposite viewpoints [S3, unofficial]. This allows training on combined original and mirrored datasets to reconstruct full 3D mesh despite lacking back-side samples [S3, unofficial]. Highly ill-posed problem; reconstruction quality for unseen parts remains lower compared to multi-view training [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the number of Gaussians and samples per ray. Training duration is 30k iterations for standard datasets and 200k for Photo Tourism [S1].
- **Memory Complexity:** Higher than standard Gaussian Splatting due to the addition of distractor Gaussians and appearance embeddings [S1].
- **Strengths:** Reduces distractor artifacts, improves background reconstruction, handles single-view symmetric objects via mirroring [S1][S3, unofficial].
- **Limitations:** Increased training time and memory usage due to distractor optimization and densification [S1]. Single-view reconstruction is ill-posed and less accurate than multi-view [S3, unofficial].
- **When to Use:** When distractor artifacts are prominent or when working with symmetric objects from limited viewpoints [S1][S3, unofficial].
- **When Not to Use:** When computational resources are limited or when high-quality multi-view data is available and distractor modeling is unnecessary [S1].

## Background
Originated from DeSplat: Decomposed Gaussian Splatting for Distractor-Free Rendering [S1]. The method extends standard Gaussian Splatting by explicitly modeling distractors. Symmetric View Mirroring was introduced in Pix2Vex for single-view reconstruction [S3, unofficial]. Perceptual loss weighting is adopted from VolSplat [S2].

## Sources
[S1] DeSplat: Decomposed Gaussian Splatting for Distractor-Free Rendering, https://doi.org/10.1109/CVPR52734.2025.00076, fetched 2026-11-03
[S2] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
[S3] Pix2Vex: Image-to-Geometry Reconstruction using a Smooth Differentiable Renderer, https://doi.org/10.48550/arxiv.1903.11149, fetched 2026-10-26 (unofficial)
