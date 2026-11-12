# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning in 3D Gaussian Splatting (3DGS) is a training phase that refines an initial set of Gaussian primitives, typically generated via a feed-forward process or coarse initialization, to achieve photorealistic novel view synthesis [S1]. It distinguishes itself from standard training by integrating densification (growing/pruning) and direct attribute optimization into a unified stage that leverages pretrained priors for faster convergence and reduced memory consumption [S1]. Unlike methods that rely solely on static densification heuristics, fine-tuning actively manages the primitive population to eliminate redundancy while preserving high-fidelity details [S1].

## How it works
### Inputs and Outputs
*   **Inputs:** An initial set of 3D Gaussian primitives $\mathcal{G} = \{g_i\}$, where each primitive $g_i$ is defined by mean position $\mu_i$, covariance matrix $\Sigma_i$ (or scale/rotation parameters), opacity $\alpha_i$, and spherical harmonics (SH) coefficients $c_i$ for color [S1]. These are typically predicted by a feed-forward network or derived from Structure-from-Motion (SfM) points [S1]. A set of training images with known camera poses [S1].
*   **Outputs:** Optimized Gaussian parameters $\mathcal{G}'$ with refined geometry, color, and opacity, resulting in a compact representation with fewer primitives than the initial set [S1].

### Data Structures
The core data structure is a list of Gaussian primitives. Each primitive $i$ stores:
*   $\mu_i \in \mathbb{R}^3$: Mean position.
*   $\sigma_i \in \mathbb{R}^3$: Scale (log-space).
*   $q_i \in \mathbb{R}^4$: Rotation (quaternion).
*   $\alpha_i \in [0, 1]$: Opacity.
*   $c_{i,0}, \dots, c_{i,d} \in \mathbb{R}^3$: SH coefficients for color (typically degree 3 or 4) [S1].

### Steps
1.  **Initialization:** Predict an initial set of 3D Gaussian primitives using a direct feed-forward process. This step generates both geometry (position, scale, rotation) and color attributes (SH coefficients) simultaneously [S1]. [G] The feed-forward network is typically a sparse convolutional network or a transformer-based architecture trained on large-scale datasets.
2.  **Differentiable Rendering:** Render training images from the current Gaussian set using differentiable rasterization. The rendered color $C$ for a pixel is computed via alpha compositing:
    $$ C = \sum_{i \in \mathcal{N}} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $\mathcal{N}$ is the set of Gaussians sorted by depth, and $c_i$ is the view-dependent color of Gaussian $i$ [S1].
3.  **Loss Computation:** Compute the total loss $\mathcal{L}$ as a weighted sum of L1 loss, SSIM loss, and entropy loss:
    $$ \mathcal{L} = (1 - \lambda_{ssim}) \mathcal{L}_{L1} + \lambda_{ssim} \mathcal{L}_{SSIM} + \lambda_e \mathcal{L}_{entropy} $$
    where $\lambda_{ssim}$ and $\lambda_e$ are hyperparameters [S1].
4.  **Optimization:** Update all geometry and color attributes using an optimizer (e.g., Adam) with a learning rate of $1 \times 10^{-3}$ [S1].
5.  **Densification and Pruning:** Enable growing and pruning of 3D primitives during fine-tuning.
    *   **Pruning:** Remove Gaussians with low opacity or small contribution to the scene to reduce redundancy and memory consumption [S1].
    *   **Growing:** Split or clone Gaussians in regions with high gradient magnitude or reconstruction error to increase density where needed [S1]. [G] Standard 3DGS densification criteria (e.g., gradient thresholding, opacity thresholding) are applied here.
6.  **Iteration:** Repeat steps 2–5 until convergence or a fixed number of iterations. The number of Gaussians is significantly reduced compared to the initial feed-forward prediction, maintaining high fidelity with lower memory usage [S1].

### Parameters
*   **Learning Rate:** $1 \times 10^{-3}$ for Adam optimizer [S1].
*   **Loss Weights:** $\lambda_{ssim}$ and $\lambda_e$ are standard 3DGS weights (typically $\lambda_{ssim} = 0.2$, $\lambda_e$ is small) [S1].
*   **Library:** `gsplat` for Gaussian rasterization [S1].

### Edge Cases and Pitfalls
*   **Redundancy:** The initial feed-forward Gaussians are often redundant. Failure to prune effectively leads to high memory consumption and slower training [S1].
*   **Convergence:** Pretrained priors accelerate convergence, but improper initialization can lead to local minima [S1].
*   **Blurry Inputs:** In defocus-blurry environments, SfM extraction is poor, leading to sparse point clouds. Supplementary point addition may be required [S1].

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

### [Augmented Radiance Field](13218-augmented-radiance-field.md)
Augmented Radiance Field is a post-enhancement framework that improves 3D Gaussian Splatting (3DGS) scenes by adding Gaussian kernels with view-dependent opacity to explicitly model specular components. It solves the problem that standard spherical harmonics (SH) cannot decouple diffuse and specular radiance, leading to poor reconstruction of sharp highlights and material artifacts. Unlike standard 3DGS which assigns SH coefficients to every primitive, this method uses distinct primitives for diffuse and specular components, avoiding the memory and stability issues of high-order SH.

### ACO-EGR Framework
ACO-EGR is a unified framework that integrates ACO and EGR into a multi-scale 3DGS backbone to jointly address scale-dependent color mismatch and primitive-allocation imbalance [S2]. The framework augments a multi-scale 3DGS backbone with two training-stage modules: ACO (Adaptive-Scale Color Offset) and EGR (Error-Guided Gaussian Reallocation) [S2]. Unlike existing methods that keep fixed color and spatial allocation after densification, ACO-EGR adapts both Gaussian color and primitive allocation together [S2]. ACO-EGR improves rendering quality without modifying the inference-stage rendering process itself; it only changes the training/optimization of the Gaussian parameters [S2].

## Operating characteristics
*   **Time Complexity:** Fine-tuning converges faster than training from scratch due to pretrained priors, but the exact speedup depends on the initial quality of the feed-forward prediction [S1].
*   **Memory Complexity:** Significantly reduced compared to the initial feed-forward prediction due to pruning, but still higher than standard 3DGS if densification is aggressive [S1].
*   **Performance:** Achieves photorealism on par with or surpassing other methods, particularly in urban view synthesis [S1].
*   **Strengths:** Fast convergence, reduced memory consumption, ability to handle redundant initial primitives [S1].
*   **Limitations:** Relies on the quality of the initial feed-forward prediction; may struggle with highly dynamic scenes or extreme occlusions without additional mechanisms [S1].
*   **When to Use:** When a coarse initialization is available (e.g., from a feed-forward network or SfM) and computational resources are limited [S1].
*   **When Not to Use:** When high-quality SfM points are unavailable and no feed-forward predictor is trained, or when real-time training is not required and standard 3DGS is sufficient [S1].

## Background
Fine-tuning in 3D Gaussian Splatting was introduced as part of the EVolSplat method, which aims to provide efficient volume-based Gaussian Splatting for urban view synthesis [S1]. The method leverages pretrained priors to accelerate network training and enable faster convergence compared to other test-time optimization methods [S1]. It addresses the challenge of redundant initial Gaussians by integrating growing and pruning mechanisms during the fine-tuning phase [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
