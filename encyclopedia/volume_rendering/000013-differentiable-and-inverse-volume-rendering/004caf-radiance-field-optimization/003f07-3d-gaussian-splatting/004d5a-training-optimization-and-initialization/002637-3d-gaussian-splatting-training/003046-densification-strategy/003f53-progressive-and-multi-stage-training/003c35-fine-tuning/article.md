# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning is a post-initialization optimization stage in 3D Gaussian Splatting (3DGS) pipelines where the geometry and color attributes of a pre-generated set of Gaussian primitives are refined to match input observations [S1]. It solves the problem of initializing a high-fidelity radiance field by decoupling the coarse generation of primitives from their precise parameter optimization [S1]. Unlike standard training which may start from sparse Structure-from-Motion (SfM) points, fine-tuning often begins with a dense, potentially redundant set of Gaussians generated via a feed-forward process or other initialization strategy, relying on the parent's principles of stage decoupling and controlled densification to prune and refine the representation [S1, S3].

## How it works
**Inputs and Outputs:**
- **Input:** A set of initialized 3D Gaussian primitives $\mathcal{G} = \{g_i\}$, where each primitive $g_i$ is defined by mean position $\mu_i$, covariance matrix $\Sigma_i$ (or scale $s_i$ and rotation $q_i$), opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_{i,lm}$ for color [S1]. These primitives are typically generated via a direct feed-forward process that predicts both geometry and color attributes [S1].
- **Output:** Optimized parameters $\mathcal{G}^*$ that minimize the rendering loss against input images, with a reduced number of primitives through pruning [S1].

**Data Structures:**
The data structure for a single Gaussian primitive $i$ consists of:
- $\mu_i \in \mathbb{R}^3$: Mean position.
- $s_i \in \mathbb{R}^3$: Scale vector (positive).
- $q_i \in \mathbb{R}^4$: Rotation quaternion (normalized).
- $\alpha_i \in [0, 1]$: Opacity.
- $\mathbf{c}_{i,lm} \in \mathbb{R}^{3 \times (L+1)^2}$: Spherical Harmonic coefficients for $L$ degrees of freedom.

**Numbered Steps:**
1. **Initialization:** Predict a set of 3D Gaussian primitives via a direct feed-forward process. This step generates initial geometry and color attributes [S1].
2. **Optimization Setup:** Initialize the optimizer (e.g., Adam) with a learning rate $\eta$ (typically $1 \times 10^{-3}$) [S1].
3. **Rendering:** For each training image, render the scene using differentiable Gaussian rasterization [S1].
4. **Loss Computation:** Compute the loss $\mathcal{L}$ between the rendered image and the ground truth. The loss typically includes:
   $$ \mathcal{L} = (1 - \lambda_{\text{ssim}}) \mathcal{L}_{\text{L1}} + \lambda_{\text{ssim}} \mathcal{L}_{\text{SSIM}} + \lambda_e \mathcal{L}_{\text{entropy}} $$
   where $\mathcal{L}_{\text{L1}}$ is the L1 loss, $\mathcal{L}_{\text{SSIM}}$ is the Structural Similarity index loss, and $\mathcal{L}_{\text{entropy}}$ is an entropy regularization term [S1].
5. **Gradient Update:** Update all geometry ($\mu_i, s_i, q_i$) and color ($\mathbf{c}_{i,lm}$) attributes using the computed gradients [S1].
6. **Densification and Pruning:** Enable growing (splitting/cloning) and pruning of 3D primitives during fine-tuning [S1]. This step is critical for reducing the number of Gaussians, as the initial feed-forward generation often produces redundant primitives [S1].
7. **Iteration:** Repeat steps 3-6 until convergence or a fixed number of iterations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fine_tuning_3dgs(initial_gaussians, images, camera_poses, num_iterations=10000):
    """
    Fine-tune 3D Gaussian Splatting from an initial set of primitives.
    
    Args:
        initial_gaussians: List of Gaussian primitives with initial parameters.
        images: List of input images.
        camera_poses: List of camera poses corresponding to images.
        num_iterations: Number of optimization steps.
        
    Returns:
        Optimized list of Gaussian primitives.
    """
    # Initialize optimizer with Adam
    optimizer = Adam(initial_gaussians.parameters(), lr=1e-3)
    
    for iteration in range(num_iterations):
        # Select a random image and its pose
        idx = random.randint(0, len(images) - 1)
        image = images[idx]
        pose = camera_poses[idx]
        
        # Render the scene using differentiable rasterization
        rendered_image = render_gaussians(initial_gaussians, pose)
        
        # Compute loss
        l1_loss = compute_l1_loss(rendered_image, image)
        ssim_loss = compute_ssim_loss(rendered_image, image)
        entropy_loss = compute_entropy_loss(initial_gaussians)
        
        # Total loss as per EVolSplat formulation
        lambda_ssim = 0.2  # Typical value, not explicitly given in S1 but standard
        lambda_e = 0.01    # Typical value, not explicitly given in S1
        loss = (1 - lambda_ssim) * l1_loss + lambda_ssim * ssim_loss + lambda_e * entropy_loss
        
        # Backpropagate and update parameters
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # Perform densification and pruning
        # This step is enabled during fine-tuning to reduce redundancy
        initial_gaussians = densify_and_prune(initial_gaussians, iteration)
        
    return initial_gaussians

def densify_and_prune(gaussians, iteration):
    """
    Enable growing and pruning of 3D primitives.
    Specific heuristics for splitting, cloning, and pruning are not detailed in S1,
    but are implied to be part of the standard 3DGS densification strategy [S1].
    [G] Standard 3DGS densification involves:
    1. Cloning Gaussians with high opacity and large view-space gradient.
    2. Splitting Gaussians with high opacity and large view-space gradient into two.
    3. Pruning Gaussians with low opacity.
    """
    # Placeholder for specific densification logic from parent article
    # The key aspect here is that this is *enabled* during fine-tuning
    # to reduce the number of Gaussians from the initial feed-forward prediction.
    return gaussians
```

**Parameters:**
- Learning rate: $1 \times 10^{-3}$ [S1].
- Optimizer: Adam [S1].
- Loss weights: $\lambda_{\text{ssim}}$ and $\lambda_e$ are used, but exact values are not provided in S1 [S1].

**Edge Cases and Pitfalls:**
- **Redundancy:** The initial feed-forward process may generate redundant Gaussians, leading to high memory consumption if not pruned [S1].
- **Convergence:** Pretrained priors accelerate training and enable faster convergence compared to test-time optimization methods [S1].

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism that dynamically adjusts the sensitivity of densification decisions based on normalized training progress, solving the trade-off between early-stage convergence speed and late-stage overfitting.

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
A learning-based, feed-forward 3D neural module that refines initial 3DGS representations to enable robust novel view synthesis from out-of-distribution (OOD) camera angles by predicting attribute residuals.

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
Reformulates 3DGS by treating Gaussians as MCMC samples, converting updates into Stochastic Gradient Langevin Dynamics (SGLD) to remove the need for manual densification heuristics.

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
A real-time deblurring framework that reconstructs sharp 3D scenes from blurry input images by manipulating covariance matrices and means using a small MLP during training.

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
A densification technique that supplements the initial point cloud with synthetic points to compensate for sparsity in defocus-blurry environments where SfM extraction is poor.

### [Gaussian Volume Mask](12710-gaussian-volume-mask.md)
A densification strategy that reduces the number of active Gaussians by applying binary masks to scale and opacity attributes based on a learnable parameter.

### [Augmented Radiance Field](13218-augmented-radiance-field.md)
A post-enhancement framework that improves 3DGS scenes by adding Gaussian kernels with view-dependent opacity to explicitly model specular components.

### [Frequency Regularization for Densification](13484-frequency-regularization-for-densification.md)
Addresses over-reconstruction by minimizing the discrepancy between the frequency spectrum of the rendered image and the ground truth in Fourier space.

### [Visibility-Based Pruning for 3D Gaussian Splatting](14019-visibility-based-pruning-for-3d-gaussian-splatting.md)
Removes occluded 3D Gaussian kernels during optimization to create a more compact, 'hard' geometry representation.

### [Error-Guided Redistribution](14037-error-guided-redistribution.md)
A family of strategies that refines primitive allocation based on reconstruction error and gradient analysis, rather than relying solely on standard adaptive density control.

### [CoherentGS](14072-coherentgs.md)
A regularized optimization approach that introduces coherency to an explicit 3D Gaussian representation by constraining Gaussian movement in 2D image space.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** The fine-tuning process significantly reduces the number of Gaussians compared to the initial feed-forward generation, thereby reducing memory consumption [S1].
- **Measured Performance:** EVolSplat achieves photorealism on par with or surpassing other methods, leveraging powerful pretrained weights [S1].
- **Scaling:** Not given in the sources.
- **Strengths:** Accelerates network training and enables faster convergence compared to other test-time optimization methods [S1]. Reduces memory consumption by pruning redundant primitives [S1].
- **Limitations:** The initial feed-forward generation may produce redundant Gaussians, requiring effective pruning strategies [S1].
- **When to use:** When starting from a dense, pre-generated set of Gaussians (e.g., from a feed-forward model) and aiming for high-fidelity reconstruction with reduced memory footprint.
- **When not:** When starting from sparse SfM points and requiring initial densification from scratch.

## Background
Fine-tuning in the context of 3D Gaussian Splatting, as described in EVolSplat, was introduced to leverage pretrained priors for efficient volume-based Gaussian Splatting for urban view synthesis [S1]. The method was developed to address the inefficiencies of standard test-time optimization by using a direct feed-forward process for initialization, followed by a fine-tuning stage that optimizes all geometry and color attributes while enabling growing and pruning [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
