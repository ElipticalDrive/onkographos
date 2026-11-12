# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning in 3D Gaussian Splatting (3DGS) refers to the optimization phase that follows an initial initialization (often via feed-forward prediction or sparse SfM), where all geometric and color attributes of the Gaussian primitives are jointly optimized to minimize rendering error [S1]. It distinguishes itself from standard training by often leveraging pretrained priors or dense initial point clouds to accelerate convergence and reduce memory consumption through aggressive pruning of redundant primitives [S1]. Unlike methods that fix the primitive count after densification, fine-tuning actively manages the Gaussian population by enabling both growing and pruning throughout the optimization process [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of initialized 3D Gaussian primitives (defined by means $\mu_i$, covariances $\Sigma_i$, opacities $\alpha_i$, and spherical harmonic coefficients $c_{i,lm}$), a set of training images with known camera poses, and a loss function configuration.
*   **Outputs:** Optimized Gaussian parameters ($\mu_i, \Sigma_i, \alpha_i, c_{i,lm}$) and a refined set of primitives with reduced redundancy.

**Data Structures:**
The core data structure is a list of $N$ Gaussian primitives. Each primitive $i$ stores:
*   Mean position $\mu_i \in \mathbb{R}^3$
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (often parameterized by rotation $R_i \in SO(3)$ and scale $s_i \in \mathbb{R}^3$)
*   Opacity $\alpha_i \in [0, 1]$
*   Spherical Harmonic coefficients $c_{i,lm}$ for view-dependent color.

**Steps:**
1.  **Initialization:** Generate an initial set of 3D Gaussian primitives. In advanced fine-tuning workflows, this is done via a direct feed-forward process that predicts both geometry and color attributes from input images or a sparse point cloud [S1].
2.  **Optimization Loop:** Iterate for a fixed number of steps or until convergence:
    a. **Sampling:** Randomly sample a subset of training views.
    b. **Rasterization:** Render the scene using differentiable Gaussian splatting to produce predicted pixel colors $\hat{C}$.
    c. **Loss Calculation:** Compute the total loss $\mathcal{L}$, typically a weighted sum of L1 loss, D-SSIM loss, and entropy loss [S1]:
       $$ \mathcal{L} = (1 - \lambda_{ssim}) \mathcal{L}_{l1} + \lambda_{ssim} \mathcal{L}_{D-SSIM} + \lambda_e \mathcal{L}_{entropy} $$
       where $\lambda_{ssim}$ and $\lambda_e$ are hyperparameters (e.g., $\lambda_{ssim} \approx 0.2$, $\lambda_e$ is small) [S1].
    d. **Backpropagation:** Compute gradients with respect to all Gaussian parameters ($\mu, \Sigma, \alpha, c$).
    e. **Update:** Update parameters using an optimizer (typically Adam with learning rate $\eta \approx 1 \times 10^{-3}$) [S1].
3.  **Densification and Pruning:** During the fine-tuning phase, actively manage the number of Gaussians:
    a. **Pruning:** Remove Gaussians with low opacity or low contribution to the scene to reduce memory usage and redundancy [S1].
    b. **Growing:** Split or clone Gaussians in regions with high gradient magnitude or high rendering error to increase detail [S1].
4.  **Convergence:** The process continues until the rendering quality plateaus or the target number of iterations is reached. The final model retains high fidelity with significantly fewer primitives than the initial feed-forward estimate [S1].

**Parameters:**
*   Learning rate: $1 \times 10^{-3}$ (Adam) [S1].
*   Loss weights: $\lambda_{ssim}$ (balance between L1 and SSIM), $\lambda_e$ (entropy regularization) [S1].
*   Optimizer: Adam [S1].

**Edge Cases and Pitfalls:**
*   **Redundancy:** Initial feed-forward Gaussians are often redundant; failing to prune them leads to high memory consumption and slower training [S1].
*   **Overfitting:** Without proper regularization (e.g., entropy loss), Gaussians may overfit to specific viewpoints, causing artifacts in novel views [S1].
*   **Precision:** Standard 32-bit floating point is typically used, but mixed precision can be employed for speed [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fine_tune_3dgs(gaussians, training_images, camera_poses, num_iterations, lr=1e-3):
    """
    Fine-tune 3D Gaussian Splatting parameters.
    
    Args:
        gaussians: List of Gaussian primitives with attributes (mean, cov, opacity, SH).
        training_images: List of input images.
        camera_poses: List of camera poses corresponding to training images.
        num_iterations: Total number of optimization steps.
        lr: Learning rate for Adam optimizer.
        
    Returns:
        Optimized gaussians.
    """
    optimizer = Adam(gaussians.parameters(), lr=lr)
    
    for step in range(num_iterations):
        # 1. Sample a random training view
        idx = random.randint(0, len(training_images) - 1)
        image = training_images[idx]
        pose = camera_poses[idx]
        
        # 2. Render the scene
        rendered_image = render_gaussians(gaussians, pose)
        
        # 3. Compute Loss
        # L = (1 - lambda_ssim) * L1 + lambda_ssim * D-SSIM + lambda_e * Entropy
        l1_loss = compute_l1_loss(rendered_image, image)
        ssim_loss = compute_dssim_loss(rendered_image, image)
        entropy_loss = compute_entropy_loss(gaussians)
        
        lambda_ssim = 0.2  # Typical value
        lambda_e = 0.001   # Typical value
        
        loss = (1 - lambda_ssim) * l1_loss + lambda_ssim * ssim_loss + lambda_e * entropy_loss
        
        # 4. Backpropagate and Update
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
        # 5. Densification and Pruning (Every N steps)
        if step % 100 == 0:
            gaussians = densify_and_prune(gaussians, threshold=0.01)
            
    return gaussians

def densify_and_prune(gaussians, threshold):
    """
    Prune low-opacity Gaussians and densify high-error regions.
    """
    # Pruning
    valid_gaussians = [g for g in gaussians if g.opacity > threshold]
    
    # Densification (simplified)
    # Identify regions with high gradient magnitude
    high_gradient_regions = identify_high_gradient_regions(gaussians)
    
    # Clone/Split Gaussians in high gradient regions
    new_gaussians = []
    for g in valid_gaussians:
        new_gaussians.append(g)
        if g in high_gradient_regions:
            new_gaussians.append(clone_gaussian(g))
            
    return new_gaussians
```

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism that uses an exponentially ascending gradient threshold to regulate splitting and cloning during fine-tuning. It allows aggressive densification in early stages when few Gaussians exist and restricts it in later stages to prevent overfitting, dynamically adjusting sensitivity based on normalized training progress.

### [Hard Gaussian Splatting](10590-hard-gaussian-splatting.md)
A strategy that identifies "hard Gaussians" (defective primitives causing blurring or needle-like artifacts) and grows them to recover missing details. It uses multi-view significant positional gradients and rendering error analysis to detect high-error regions, refining the Gaussian distribution during fine-tuning to resolve artifacts missed by standard adaptive density control.

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
A feed-forward module that refines initial 3DGS representations to enable robust novel view synthesis from out-of-distribution (OOD) camera angles. It applies a point transformer network to predict attribute residuals, addressing the bias of standard 3DGS optimization which overfits to input views and fails to generalize to unseen high-elevation perspectives.

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
Reformulates 3DGS by treating Gaussians as MCMC samples from an underlying probability distribution. It converts updates into Stochastic Gradient Langevin Dynamics (SGLD) by adding noise, removing the need for manual densification heuristics like cloning and opacity resetting during fine-tuning.

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
A real-time framework that reconstructs sharp scenes from blurry inputs by manipulating covariance matrices and means using a small MLP during fine-tuning. It handles defocus and motion blur via differentiable rasterization and K-nearest-neighbor interpolation to address sparse point clouds.

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
A densification technique that supplements the initial point cloud with synthetic points to compensate for sparsity in far-end regions where SfM extraction is poor. It improves reconstruction fidelity in large-depth-of-field scenarios during the fine-tuning phase.

### [Gaussian Volume Mask](12710-gaussian-volume-mask.md)
A strategy that reduces active Gaussians by applying binary masks to scale and opacity attributes based on a learnable parameter. It addresses excessive memory requirements by consistently masking out unessential Gaussians throughout the entire fine-tuning process, considering both geometric volume and transparency.

### [Error-Guided Gaussian Reallocation](13012-error-guided-gaussian-reallocation.md)
A module that redistributes a fixed budget of Gaussians toward regions with persistent coarse-scale reconstruction errors without increasing the total count. It moves low-contribution Gaussians to high-error regions using contribution- and error-aware selection, refining post-densification allocation.

### [Augmented Radiance Field](13218-augmented-radiance-field.md)
A post-enhancement framework that adds Gaussian kernels with view-dependent opacity to explicitly model specular components. It solves the problem of standard SH failing to decouple diffuse and specular radiance, using distinct primitives for each component to avoid memory and stability issues.

### ACO-EGR Framework
A unified framework integrating Adaptive-Scale Color Offset (ACO) and Error-Guided Gaussian Reallocation (EGR) into a multi-scale 3DGS backbone [S2]. It jointly addresses scale-dependent color mismatch and primitive-allocation imbalance by adapting per-Gaussian color across scales and redistributing Gaussians toward high-error regions, improving multi-scale rendering quality without modifying the inference process [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of Gaussians $N$ and the number of training views per iteration. Fine-tuning with feed-forward initialization converges faster than random initialization [S1].
*   **Memory Complexity:** Proportional to $N \times (3 + 3 + 1 + C)$, where $C$ is the number of SH coefficients. Pruning during fine-tuning significantly reduces memory consumption compared to the initial feed-forward estimate [S1].
*   **Performance:** Achieves photorealism on par with or surpassing other methods, leveraging pretrained weights for faster convergence [S1].
*   **Strengths:** Accelerated training, reduced memory footprint, high fidelity.
*   **Limitations:** Requires careful tuning of densification/pruning thresholds to avoid overfitting or underfitting [S1].

## Background
Fine-tuning in 3DGS was introduced as a method to improve efficiency and quality by leveraging initial estimates from feed-forward networks or dense SfM point clouds [S1]. It builds on the original 3DGS training pipeline but emphasizes the reduction of redundant primitives and the acceleration of convergence through pretrained priors [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
