# Fine-tuning

Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning is a post-initialization optimization stage in 3D Gaussian Splatting (3DGS) where a pre-generated set of Gaussian primitives is refined to achieve photorealistic novel view synthesis. It solves the problem of initializing a scene representation with high-fidelity priors (e.g., from a feed-forward network or sparse SfM) by directly optimizing all geometric and color attributes. It sets itself apart from static initialization methods by enabling dynamic **growing and pruning** of primitives during the optimization phase, which significantly reduces memory consumption by removing redundant initial Gaussians while maintaining high fidelity [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of initialized 3D Gaussian primitives (means, covariances, opacities, spherical harmonics coefficients) generated via a feed-forward process or sparse point cloud.
*   **Output:** Optimized 3D Gaussian primitives with refined geometry and color attributes.

**Data Structures:**
The data structure is identical to standard 3DGS:
*   **Means ($\mu$):** 3D position vector $(x, y, z)$.
*   **Covariances ($\Sigma$):** Represented by a rotation matrix $R$ and a scaling vector $s$, or directly as a 3x3 symmetric positive-definite matrix.
*   **Opacity ($\alpha$):** Scalar value in $[0, 1]$.
*   **Spherical Harmonics (SH):** Coefficients for view-dependent color.

**Numbered Steps:**
1.  **Initialization:** Predict a set of 3D Gaussian primitives for initialization via a direct feed-forward process where both geometry and color attributes are generated [S1].
2.  **Optimization Setup:** Initialize the optimizer (e.g., Adam) with a learning rate of $1 \times 10^{-3}$ [S1].
3.  **Differentiable Rendering:** Render the scene using differentiable Gaussian rasterization (e.g., via `gsplat`) [S1].
4.  **Loss Computation:** Compute the total loss $\mathcal{L}$, typically including SSIM and L1 loss, plus an entropy regularizer:
    $$ \mathcal{L} = \mathcal{L}_{\text{ssim}} + \lambda_e \mathcal{L}_{\text{entropy}} $$
    where $\lambda_e$ is a weighting factor for the entropy term [S1].
5.  **Gradient Update:** Update all geometry and color attributes directly using the computed gradients [S1].
6.  **Densification Management:** Enable growing (splitting/cloning) and pruning of 3D primitives during fine-tuning based on gradient magnitude and opacity thresholds [S1].
7.  **Iteration:** Repeat steps 3-6 until convergence.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize Gaussians from feed-forward prediction or SfM
gaussians = initialize_gaussians(input_data)

# Setup optimizer
optimizer = Adam(gaussians.parameters(), lr=1e-3)

# Setup rasterizer
rasterizer = GaussianRasterizer()

for step in range(max_steps):
    # Sample a random camera view
    camera = sample_camera_view()
    
    # Render the scene
    rendered_image = rasterizer.render(gaussians, camera)
    
    # Compute loss
    # Note: Ground truth image 'gt_image' is assumed available
    loss_ssim = compute_ssim(rendered_image, gt_image)
    loss_l1 = compute_l1(rendered_image, gt_image)
    loss_entropy = compute_entropy(gaussians.opacity)
    
    total_loss = loss_ssim + loss_l1 + lambda_e * loss_entropy
    
    # Backpropagate and update
    optimizer.zero_grad()
    total_loss.backward()
    optimizer.step()
    
    # Perform growing and pruning
    # This step is specific to the fine-tuning phase to manage density
    gaussians = manage_density(gaussians, step)

def manage_density(gaussians, step):
    """
    Enables growing and pruning of 3D primitives during fine-tuning.
    Reduces the number of Gaussians by removing redundant ones.
    """
    # Prune low-opacity Gaussians
    gaussians = prune_low_opacity(gaussians, threshold=0.005)
    
    # Split/Clone high-gradient Gaussians
    gaussians = densify_gaussians(gaussians, threshold=gradient_threshold)
    
    return gaussians
```

**Parameters:**
*   **Learning Rate:** $1 \times 10^{-3}$ [S1].
*   **Optimizer:** Adam [S1].
*   **$\lambda_e$:** Weight for entropy loss (value not specified in sources).
*   **Pruning Threshold:** Not given in the sources.
*   **Densification Threshold:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Redundancy:** The initial feed-forward Gaussians are often redundant, leading to high memory usage if not pruned [S1].
*   **Convergence:** Pretrained priors accelerate training, but improper fine-tuning may lead to overfitting or artifacts if densification is not managed [S1].

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism that uses an exponentially ascending gradient threshold to regulate splitting and cloning, solving the trade-off between early-stage convergence speed and late-stage overfitting.

### [Hard Gaussian Splatting](10590-hard-gaussian-splatting.md)
Identifies "hard Gaussians" causing blurring or needle-like artifacts and grows them to recover missing details, using multi-view significant positional gradients.

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
A learning-based, feed-forward module that refines initial 3DGS representations to enable robust novel view synthesis from out-of-distribution camera angles.

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
Reformulates 3DGS as MCMC samples, converting updates into Stochastic Gradient Langevin Dynamics to remove the need for manual densification heuristics.

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
A real-time deblurring framework that reconstructs sharp scenes from blurry inputs by manipulating covariance matrices and means using an MLP.

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
A densification technique that supplements initial point clouds with synthetic points to compensate for sparsity in defocus-blurry environments.

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians and the cost of differentiable rasterization. Fine-tuning accelerates convergence compared to test-time optimization methods due to pretrained priors [S1].
*   **Memory Complexity:** Significantly reduced during fine-tuning as redundant initial Gaussians are pruned, maintaining high fidelity with lower memory consumption [S1].
*   **Performance:** Achieves photorealism on par with or surpassing other methods [S1].
*   **Strengths:** Fast convergence, memory efficiency through pruning, high fidelity.
*   **Limitations:** Relies on the quality of the initial feed-forward prediction or SfM points.
*   **When to use:** When a high-quality initial representation is available and memory efficiency is a concern.
*   **When not to use:** When the initial initialization is poor, as fine-tuning may not recover from significant structural errors.

## Background
Fine-tuning in the context of 3D Gaussian Splatting was introduced in **EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis** [S1]. The method leverages pretrained weights to accelerate network training and enable faster convergence compared to other test-time optimization methods. It addresses the inefficiency of standard 3DGS initialization by dynamically managing the Gaussian population during the optimization phase.

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
