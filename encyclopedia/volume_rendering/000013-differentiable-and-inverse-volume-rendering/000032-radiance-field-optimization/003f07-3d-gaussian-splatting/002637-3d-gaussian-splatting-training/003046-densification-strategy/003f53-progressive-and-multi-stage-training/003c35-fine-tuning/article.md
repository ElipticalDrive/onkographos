# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning in 3D Gaussian Splatting (3DGS) refers to the phase where a pre-initialized set of Gaussian primitives—often generated via a feed-forward network or coarse initialization—is optimized to achieve high-fidelity novel view synthesis [S1]. It solves the problem of initializing a dense, accurate scene representation from sparse or noisy inputs by leveraging pretrained priors to accelerate convergence and reduce memory consumption compared to training from scratch [S1]. It sets itself apart from standard training by explicitly enabling growing and pruning of primitives during the optimization process, allowing the model to discard redundant initial Gaussians while refining geometry and color attributes directly [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of initial 3D Gaussian primitives $\{G_i\}$ with initial geometry (mean $\mu_i$, covariance $\Sigma_i$ or scale/rotation) and color attributes (SH coefficients $c_i$), typically predicted by a feed-forward process [S1]. Initial camera poses and intrinsics from Structure-from-Motion (SfM) [S1].
*   **Output:** Optimized Gaussian parameters $\{\mu_i, \Sigma_i, c_i, \alpha_i\}$ and a pruned set of active Gaussians [S1].

**Data Structures:**
*   **Gaussian Primitive:** Defined by mean position $\mu_i \in \mathbb{R}^3$, scale $s_i \in \mathbb{R}^3$, rotation $q_i \in \mathbb{R}^4$ (unit quaternion), opacity $\alpha_i \in [0,1]$, and spherical harmonic coefficients $c_i$ [S1].
*   **Optimization State:** Maintained by an optimizer (e.g., Adam) [S1].

**Steps:**
1.  **Initialization:** Predict a set of 3D Gaussian primitives for initialization via a direct feed-forward process where both geometry and color attributes are generated [S1].
2.  **Optimization Setup:** Initialize the optimizer (e.g., Adam) with a learning rate (typically $1 \times 10^{-3}$) [S1].
3.  **Iterative Optimization:** For each training iteration:
    a. Render images from random camera viewpoints using differentiable rasterization [S1].
    b. Compute loss between rendered and ground-truth images. The loss function typically includes photometric loss, SSIM loss, and entropy loss:
       $$ \mathcal{L} = \mathcal{L}_{\text{photometric}} + \lambda_{\text{ssim}} \mathcal{L}_{\text{ssim}} + \lambda_e \mathcal{L}_{\text{entropy}} $$
       where $\lambda_{\text{ssim}}$ and $\lambda_e$ are weighting coefficients [S1].
    c. Backpropagate gradients to update all geometry and color attributes directly, following the vanilla 3DGS optimization scheme [S1].
4.  **Growing and Pruning:** Enable growing (splitting/cloning) and pruning (removing low-opacity Gaussians) of 3D primitives during fine-tuning [S1]. This step is critical as the initial feed-forward Gaussians are often redundant; pruning significantly reduces the number of Gaussians while maintaining high fidelity [S1].
5.  **Convergence:** Continue until the model achieves photorealism, leveraging pretrained priors for faster convergence [S1].

**Parameters:**
*   **Learning Rate:** $1 \times 10^{-3}$ (Adam optimizer) [S1].
*   **Loss Weights:** $\lambda_{\text{ssim}}$ and $\lambda_e$ are used but specific values are not given in the sources [S1].
*   **Library:** `gsplat` for Gaussian rasterization [S1].

**Edge Cases and Pitfalls:**
*   **Redundancy:** The initial feed-forward Gaussians are redundant, leading to high memory consumption if not pruned [S1].
*   **Convergence Speed:** While pretrained priors accelerate training, improper initialization may lead to suboptimal local minima if fine-tuning is not performed correctly [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Initialize Gaussians from feed-forward prediction
gaussians = feed_forward_predict_geometry_and_color(scene_data)

# Initialize optimizer
optimizer = Adam(gaussians.parameters(), lr=1e-3)

# Training loop
for iteration in range(max_iterations):
    # Sample random camera
    camera = sample_random_camera()
    
    # Render image
    rendered_image = render(gaussians, camera)
    
    # Compute loss
    gt_image = get_ground_truth(camera)
    loss = compute_loss(rendered_image, gt_image)
    # Loss components: photometric, SSIM, entropy
    # L = L_photometric + lambda_ssim * L_ssim + lambda_e * L_entropy
    
    # Backpropagate and update
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    # Prune and grow Gaussians
    gaussians = prune_and_grow(gaussians, threshold=opacity_threshold)
```

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism for 3D Gaussian Splatting that uses an exponentially ascending gradient threshold to regulate the splitting and cloning of Gaussians during training. It solves the trade-off between early-stage convergence speed and late-stage overfitting by allowing aggressive densification when few Gaussians exist and restricting it as the scene becomes dense. It sets itself apart from static threshold methods by dynamically adjusting the sensitivity of densification decisions based on the normalized training progress.

### [Hard Gaussian Splatting](10590-hard-gaussian-splatting.md)
Hard Gaussian Splatting (HGS) is a densification strategy for 3D Gaussian Splatting that identifies "hard Gaussians"—defective primitives causing blurring or needle-like artifacts—and grows them to recover missing scene details. Unlike standard 3DGS, which relies on averaged view-space positional gradients that smooth out local errors and hinder the growth of defective Gaussians, HGS uses multi-view significant positional gradients (PGHGS) and rendering error analysis (REHGS) to detect regions with high reconstruction error. It builds on [Fine-tuning](9138-fine-tuning.md) by refining the Gaussian distribution during the training phase to resolve artifacts that standard adaptive density control misses.

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
SplatFormer is a learning-based, feed-forward 3D neural module that refines initial 3D Gaussian Splatting (3DGS) representations to enable robust novel view synthesis from out-of-distribution (OOD) camera angles. It addresses the bias of standard 3DGS optimization, which overfits to input views and fails to generalize to unseen high-elevation perspectives, by applying a point transformer network that predicts attribute residuals. Unlike 2D image restoration methods that suffer from multi-view inconsistencies, SplatFormer operates directly on 3D primitives, ensuring geometric consistency while recovering details lost in OOD views.

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
This method reformulates 3D Gaussian Splatting (3DGS) by treating the set of 3D Gaussians as random samples drawn from an underlying probability distribution describing the scene, specifically as Markov Chain Monte Carlo (MCMC) samples. It converts conventional 3DGS updates into Stochastic Gradient Langevin Dynamics (SGLD) updates by introducing a noise term to promote sample exploration, thereby removing the need for manual densification heuristics like cloning, splitting, and opacity resetting. Unlike sibling methods that rely on complex gradient thresholds or hard constraints, this approach relies on probabilistic state transitions and regularization to manage Gaussian population.

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
Deblurring 3D Gaussian Splatting is a real-time deblurring framework that reconstructs sharp 3D scenes from blurry input images by manipulating the covariance matrices and means of 3D Gaussians using a small Multi-Layer Perceptron (MLP) during training. Unlike volumetric rendering-based methods like Deblur-NeRF which require seconds per image, this method uses differentiable rasterization to achieve real-time inference speeds (>800 FPS) while handling both defocus and camera motion blur. It addresses the sparse point clouds caused by blurry inputs through K-nearest-neighbor interpolation and depth-based pruning.

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
A densification technique designed to compensate for the sparsity of 3D Gaussians at the far end of scenes in defocus-blurry environments, where Structure-from-Motion (SfM) extraction is typically poor. It supplements the initial point cloud with synthetic points to ensure adequate coverage in regions where SfM fails to extract features, thereby improving reconstruction fidelity in large-depth-of-field scenarios.

### [Gaussian Volume Mask](12710-gaussian-volume-mask.md)
Gaussian Volume Mask is a densification strategy for 3D Gaussian Splatting that reduces the number of active Gaussians by applying binary masks to scale and opacity attributes based on a learnable parameter. It addresses the excessive memory and storage requirements caused by the proliferation of Gaussians during training, which often include redundant primitives with negligible volume or transparency. Unlike original 3DGS, which relies solely on opacity-based control and stops densifying in the middle of training, this method consistently masks out unessential Gaussians throughout the entire process by considering both geometric volume and transparency.

### [Error-Guided Gaussian Reallocation](13012-error-guided-gaussian-reallocation.md)
Error-Guided Gaussian Reallocation (EGR) is a training-stage module that redistributes a fixed budget of Gaussian primitives toward regions with persistent coarse-scale reconstruction errors, without increasing the total number of Gaussians. It refines post-densification Gaussian allocation by moving low-contribution (LC) Gaussians to high-error (HE) regions within a fixed Gaussian set, using contribution- and error-aware selection. Unlike methods using explicit rollback or separate re-correction modules, EGR uses adaptive relocation count, parameter reset, and continued optimization as its stabilization strategy.

### ACO-EGR Framework
ACO-EGR is a unified framework that integrates ACO and EGR into a multi-scale 3DGS backbone to jointly address scale-dependent color mismatch and primitive-allocation imbalance [S2]. The framework augments a multi-scale 3DGS backbone with two training-stage modules: ACO and EGR [S2]. Unlike existing methods that keep fixed color and spatial allocation after densification, ACO-EGR adapts both Gaussian color and primitive allocation together [S2]. ACO-EGR improves rendering quality without modifying the inference-stage rendering process itself; it only changes the training/optimization of the Gaussian parameters [S2].

## Operating characteristics
*   **Time Complexity:** Faster convergence compared to training from scratch due to pretrained priors [S1].
*   **Memory Complexity:** Significantly reduced memory consumption after pruning redundant initial Gaussians [S1].
*   **Performance:** Achieves photorealism on par with or surpassing other methods [S1].
*   **Strengths:** Accelerated training, reduced memory footprint, high fidelity [S1].
*   **Limitations:** Relies on the quality of the initial feed-forward prediction [S1].
*   **When to Use:** When high-fidelity synthesis is required with limited training time or memory, and a good initialization method is available [S1].
*   **When Not to Use:** When no pretrained model or initialization method is available, or when the scene is too complex for the feed-forward predictor to handle [S1].

## Background
Fine-tuning in 3D Gaussian Splatting was introduced in the context of efficient volume-based Gaussian Splatting for urban view synthesis [S1]. It leverages pretrained priors to accelerate network training and enable faster convergence compared to other test-time optimization methods [S1]. The method is implemented using Adam optimizer and libraries like `torchsparse` and `gsplat` [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
