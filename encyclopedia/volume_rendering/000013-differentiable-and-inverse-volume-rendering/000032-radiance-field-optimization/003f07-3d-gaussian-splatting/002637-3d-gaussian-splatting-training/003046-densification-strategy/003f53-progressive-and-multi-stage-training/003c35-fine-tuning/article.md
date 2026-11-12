# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning in 3D Gaussian Splatting (3DGS) refers to the optimization phase where a pre-initialized set of 3D Gaussian primitives is refined to achieve high-fidelity novel view synthesis. It solves the problem of initializing a dense, redundant Gaussian cloud by leveraging pretrained priors or feed-forward predictions to accelerate convergence and reduce memory consumption compared to training from scratch [S1]. It sets itself apart from standard training by starting with a structured initialization (often from a feed-forward network) and actively managing the Gaussian population through growing and pruning during the optimization process, rather than relying solely on static densification heuristics [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of initial 3D Gaussian primitives $\{G_i\}$ predicted by a feed-forward process, containing geometry (mean $\mu_i$, covariance $\Sigma_i$ or scale/rotation) and color attributes (Spherical Harmonics coefficients $c_i$); a set of training images with known camera poses.
*   **Outputs:** Optimized geometry and color attributes for the Gaussian primitives.

**Data Structures:**
The Gaussian primitives are stored in a list or array where each element $i$ contains:
*   $\mu_i \in \mathbb{R}^3$: Mean position.
*   $\sigma_i \in \mathbb{R}^3$: Scale (diagonal of covariance).
*   $q_i \in \mathbb{R}^4$: Rotation (quaternion).
*   $c_i$: Spherical Harmonics (SH) coefficients for color.
*   $\alpha_i \in [0, 1]$: Opacity.

**Steps:**
1.  **Initialization:** Predict a set of 3D Gaussian primitives via a direct feed-forward process. Both geometry and color attributes are generated simultaneously [S1]. This step provides a coarse but structurally sound initialization, often resulting in a redundant cloud of Gaussians [S1].
2.  **Optimization Setup:** Initialize the optimizer (e.g., Adam) with a learning rate (typically $1 \times 10^{-3}$) [S1]. The loss function combines standard 3DGS terms:
    $$ \mathcal{L} = (1 - \lambda_{\text{ssim}}) \mathcal{L}_1 + \lambda_{\text{ssim}} \mathcal{L}_{\text{SSIM}} + \lambda_e \mathcal{L}_{\text{entropy}} $$
    where $\mathcal{L}_1$ is the L1 photometric loss, $\mathcal{L}_{\text{SSIM}}$ is the structural similarity loss, and $\mathcal{L}_{\text{entropy}}$ is an entropy regularization term [S1].
3.  **Iterative Refinement:** Optimize all geometry and color attributes directly following the vanilla 3DGS rendering and gradient update pipeline [S1].
4.  **Population Control:** Enable growing (splitting/cloning) and pruning of 3D primitives during fine-tuning [S1]. This step is critical because the initial feed-forward Gaussians are often redundant. The pruning mechanism significantly reduces the number of Gaussians, maintaining high fidelity while lowering memory consumption [S1].
5.  **Convergence:** Continue optimization until convergence. The pretrained priors accelerate network training and enable faster convergence compared to test-time optimization methods without priors [S1].

**Parameters:**
*   Learning rate: $1 \times 10^{-3}$ [S1].
*   Optimizer: Adam [S1].
*   SH Degree: Not specified in sources, but implied by standard 3DGS implementation [S1].

**Edge Cases and Pitfalls:**
*   **Redundancy:** The initial feed-forward prediction may produce a large number of redundant Gaussians. Without active pruning, this leads to excessive memory usage and slower rendering [S1].
*   **Overfitting:** Fine-tuning from a pretrained model risks overfitting to the input views if the regularization terms (like entropy) are not balanced correctly [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fine_tune_3dgs(initial_gaussians, training_images, camera_poses, num_iterations):
    """
    Fine-tunes 3D Gaussian Splatting from a feed-forward initialization.
    
    Args:
        initial_gaussians: List of Gaussian objects with mean, scale, rotation, color, opacity.
        training_images: List of target RGB images.
        camera_poses: List of camera pose matrices corresponding to training_images.
        num_iterations: Number of optimization steps.
        
    Returns:
        Optimized list of Gaussian objects.
    """
    # Initialize optimizer with Adam
    optimizer = Adam(params=initial_gaussians.parameters(), lr=1e-3)
    
    for step in range(num_iterations):
        # Sample a random training image and its pose
        img_idx = random.randint(0, len(training_images) - 1)
        target_img = training_images[img_idx]
        pose = camera_poses[img_idx]
        
        # Render the scene using current Gaussians
        rendered_img = render_gaussians(initial_gaussians, pose)
        
        # Compute loss
        l1_loss = compute_l1_loss(rendered_img, target_img)
        ssim_loss = compute_ssim_loss(rendered_img, target_img)
        entropy_loss = compute_entropy_loss(initial_gaussians)
        
        # Combined loss as per [S1]
        lambda_ssim = 0.2  # Typical value, not explicitly given but standard
        lambda_e = 0.001   # Typical value, not explicitly given but standard
        loss = (1 - lambda_ssim) * l1_loss + lambda_ssim * ssim_loss + lambda_e * entropy_loss
        
        # Backpropagate and update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Prune redundant Gaussians
        # Note: Specific pruning criteria (e.g., opacity threshold) are not detailed in [S1]
        # but are implied by "enable growing and pruning" [S1].
        # [G] Standard 3DGS pruning: remove Gaussians with opacity < threshold (e.g., 0.005)
        prune_gaussians(initial_gaussians, opacity_threshold=0.005)
        
        # Densify if necessary (growing)
        # [G] Standard 3DGS densification: split/clone Gaussians with high gradient magnitude
        densify_gaussians(initial_gaussians, gradient_threshold=0.0002)
        
    return initial_gaussians
```

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism that dynamically adjusts the sensitivity of densification decisions based on normalized training progress, allowing aggressive densification early and restricting it later.

### [Hard Gaussian Splatting](10590-hard-gaussian-splatting.md)
Identifies "hard Gaussians" causing artifacts and grows them using multi-view significant positional gradients and rendering error analysis, refining the distribution during training.

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
A learning-based feed-forward module that refines initial 3DGS representations by predicting attribute residuals to handle out-of-distribution camera angles.

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
Reformulates 3DGS updates as Stochastic Gradient Langevin Dynamics (SGLD) to remove manual densification heuristics by introducing noise for sample exploration.

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
Uses a small MLP to manipulate covariance and means during training to reconstruct sharp scenes from blurry inputs, handling defocus and motion blur.

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
Supplements initial point clouds with synthetic points to compensate for sparsity in defocus-blurry environments where SfM extraction is poor.

### [Gaussian Volume Mask](12710-gaussian-volume-mask.md)
Reduces active Gaussians by applying binary masks to scale and opacity based on a learnable parameter, considering both geometric volume and transparency.

### [Error-Guided Gaussian Reallocation](13012-error-guided-gaussian-reallocation.md)
Redistributes a fixed budget of Gaussians toward regions with persistent coarse-scale reconstruction errors without increasing the total number.

### [Augmented Radiance Field](13218-augmented-radiance-field.md)
Adds Gaussian kernels with view-dependent opacity to model specular components, decoupling diffuse and specular radiance.

### ACO-EGR Framework
ACO-EGR is a unified framework that integrates Adaptive-Scale Color Offset (ACO) and Error-Guided Gaussian Reallocation (EGR) into a multi-scale 3DGS backbone to jointly address scale-dependent color mismatch and primitive-allocation imbalance [S2]. It augments the backbone with two training-stage modules: ACO adapts per-Gaussian color across rendering scales through scale-specific DC color offsets, and EGR refines post-densification allocation by moving low-contribution Gaussians to high-error regions within a fixed Gaussian set [S2]. Unlike existing methods that keep fixed color and spatial allocation after densification, ACO-EGR adapts both together, improving multi-scale rendering quality without modifying the inference-stage rendering process [S2].

## Operating characteristics
*   **Time Complexity:** Faster convergence compared to training from scratch due to pretrained priors [S1]. The exact time per iteration is not specified, but the pruning step reduces the number of Gaussians, potentially speeding up rendering and subsequent iterations [S1].
*   **Memory Complexity:** Significantly reduced memory consumption compared to the initial feed-forward prediction due to pruning of redundant primitives [S1].
*   **Measured Performance:** Achieves photorealism on par with or surpassing other methods on urban view synthesis tasks [S1].
*   **Strengths:** Accelerates training, reduces memory usage, handles redundant initializations.
*   **Limitations:** Relies on the quality of the initial feed-forward prediction; if the initialization is poor, fine-tuning may struggle to converge to a high-quality solution.
*   **When to use:** When computational resources for training from scratch are limited, or when a coarse initialization is available (e.g., from a pretrained network or SfM).
*   **When not to use:** When the initial prediction is highly inaccurate or when real-time inference speed is the primary constraint and the overhead of pruning/densification is unacceptable.

## Background
Fine-tuning in the context of 3D Gaussian Splatting was introduced in the EVolSplat method, which proposed using a direct feed-forward process to predict initial 3D Gaussian primitives for urban view synthesis [S1]. This approach leverages pretrained priors to accelerate network training and enable faster convergence compared to test-time optimization methods without such priors [S1]. The method was published in 2025 [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
