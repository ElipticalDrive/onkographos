# Fine-tuning

Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning in 3D Gaussian Splatting (3DGS) is a post-initialization optimization phase where the parameters of a pre-generated set of 3D Gaussian primitives are refined to achieve photorealistic novel view synthesis [S1]. It addresses the trade-off between reconstruction fidelity and memory efficiency by allowing the growing and pruning of primitives during the optimization process, thereby removing redundant Gaussians introduced by the initial feed-forward prediction [S1]. It sets itself apart from standard training by leveraging pretrained priors to accelerate convergence and by explicitly managing the Gaussian population size through densification heuristics during the refinement stage [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of initial 3D Gaussian primitives $\mathcal{G}_0 = \{\mathbf{g}_i\}$ generated via a direct feed-forward process, where each primitive $\mathbf{g}_i$ contains geometry attributes (mean $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$ or scale/rotation) and color attributes (spherical harmonics coefficients $\mathbf{c}_i$) [S1].
*   **Output:** An optimized set of 3D Gaussians $\mathcal{G}_{final}$ with refined geometry and color attributes, and a reduced count of primitives compared to $\mathcal{G}_0$ [S1].

**Data Structures:**
The Gaussian cloud is represented as a list or array of primitives. Each primitive stores:
*   Mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$
*   Scale $s_i \in \mathbb{R}^3$ and Rotation $r_i \in \mathbb{R}^4$ (quaternion)
*   Opacity $\alpha_i \in [0, 1]$
*   Spherical Harmonics coefficients $\mathbf{c}_i$ for color representation [S1].

**Numbered Steps:**
1.  **Initialization:** Predict the initial set of 3D Gaussian primitives $\mathcal{G}_0$ using a feed-forward network. Both geometry and color attributes are generated simultaneously [S1].
2.  **Optimization Setup:** Initialize the optimizer (e.g., Adam) with a learning rate of $1 \times 10^{-3}$ [S1].
3.  **Iterative Refinement:** For each training iteration:
    a. Render the scene using differentiable rasterization (e.g., via `gsplat`) [S1].
    b. Compute the loss between the rendered image and the ground truth image. The loss typically includes SSIM and entropy terms, as defined in the EVolSplat formulation: $\mathcal{L} = \mathcal{L}_{SSIM} + \lambda_e \mathcal{L}_{entropy}$ [S1].
    c. Compute gradients with respect to all geometry and color attributes [S1].
    d. Update all parameters using the optimizer [S1].
4.  **Densification Control:** Enable growing (cloning/splitting) and pruning of 3D primitives during the fine-tuning phase [S1]. This step is critical for reducing the number of Gaussians, as the initial feed-forward predictions are often redundant [S1].
5.  **Convergence:** Repeat steps 3-4 until convergence. The pretrained priors accelerate this process compared to test-time optimization methods without priors [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fine_tuning_3dgs(initial_gaussians, images, camera_poses, num_iterations):
    """
    Fine-tunes 3D Gaussian Splatting primitives initialized by a feed-forward model.
    
    Args:
        initial_gaussians: List of Gaussian primitives with geometry and color attributes.
        images: List of ground truth images.
        camera_poses: List of camera poses corresponding to images.
        num_iterations: Number of optimization steps.
        
    Returns:
        Optimized list of Gaussian primitives.
    """
    # Initialize optimizer with Adam, lr=1e-3
    optimizer = Adam(params=initial_gaussians.parameters(), lr=1e-3)
    
    gaussians = initial_gaussians
    
    for i in range(num_iterations):
        # Sample a random image and pose
        idx = random.randint(0, len(images) - 1)
        gt_image = images[idx]
        pose = camera_poses[idx]
        
        # Render the scene
        rendered_image = render(gaussians, pose)
        
        # Compute loss (SSIM + Entropy)
        # Note: Specific lambda_e value not given in sources [S1]
        loss = compute_ssim_loss(rendered_image, gt_image) + lambda_e * compute_entropy_loss(gaussians)
        
        # Backpropagate and update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Enable growing and pruning during fine-tuning
        # This reduces the number of Gaussians by removing redundant ones
        gaussians = densification_step(gaussians)
        
    return gaussians
```

**Parameters:**
*   Learning Rate: $1 \times 10^{-3}$ [S1].
*   Optimizer: Adam [S1].
*   Lambda for Entropy Loss ($\lambda_e$): Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Redundancy:** The initial feed-forward Gaussians are redundant, leading to high memory consumption if pruning is not enabled [S1].
*   **Convergence Speed:** Without pretrained priors, convergence is slower [S1].

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism that dynamically adjusts the sensitivity of densification decisions based on normalized training progress, solving the trade-off between early-stage convergence speed and late-stage overfitting.

### [Hard Gaussian Splatting](10590-hard-gaussian-splatting.md)
Identifies "hard Gaussians" causing artifacts and grows them using multi-view significant positional gradients and rendering error analysis, refining the distribution during training.

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
A feed-forward module that predicts attribute residuals to refine 3DGS representations, enabling robust novel view synthesis from out-of-distribution camera angles.

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
Reformulates 3DGS updates as Stochastic Gradient Langevin Dynamics (SGLD) to remove the need for manual densification heuristics like cloning and splitting.

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
Reconstructs sharp scenes from blurry inputs by manipulating covariance matrices and means using an MLP, achieving real-time inference speeds.

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
Supplements initial point clouds with synthetic points to compensate for sparsity in defocus-blurry environments where SfM extraction is poor.

### [Gaussian Volume Mask](12710-gaussian-volume-mask.md)
Reduces active Gaussians by applying binary masks to scale and opacity attributes based on a learnable parameter, addressing excessive memory requirements.

## Operating characteristics
*   **Time Complexity:** Accelerated convergence compared to test-time optimization methods without pretrained priors [S1].
*   **Memory Complexity:** Significantly reduced memory consumption compared to the initial feed-forward prediction due to pruning of redundant Gaussians [S1].
*   **Performance:** Achieves photorealism on par with or surpassing other methods [S1].
*   **Strengths:** High fidelity, reduced memory usage, faster convergence due to pretrained priors [S1].
*   **Limitations:** Relies on the quality of the initial feed-forward prediction; requires densification heuristics to manage population size [S1].
*   **When to Use:** When high-fidelity novel view synthesis is required with limited memory resources, and a pretrained feed-forward model is available for initialization [S1].
*   **When Not to Use:** When no pretrained priors are available, or when the initial feed-forward prediction is of very poor quality [S1].

## Background
Fine-tuning in the context of 3D Gaussian Splatting was introduced in the EVolSplat method for efficient volume-based Gaussian Splatting for urban view synthesis [S1]. The method leverages pretrained weights to accelerate network training and enable faster convergence compared to other test-time optimization methods [S1]. It builds on the standard 3DGS training pipeline by integrating a feed-forward initialization step and enabling growing and pruning during the fine-tuning phase [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
