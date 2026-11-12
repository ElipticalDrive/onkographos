# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning in 3D Gaussian Splatting (3DGS) is a training phase that refines an initial set of Gaussian primitives to achieve photorealistic novel view synthesis, often leveraging pretrained weights or feed-forward initialization to accelerate convergence [S1]. It distinguishes itself from standard training by decoupling the initialization of geometry and color from the subsequent optimization, allowing for aggressive densification (growing and pruning) during the refinement stage to correct redundancies and improve fidelity [S1]. This approach builds on [Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md) by utilizing a coarse initialization to enable faster convergence and reduced memory consumption compared to optimizing from scratch.

## How it works
**Inputs and Outputs:**
*   **Input:** An initial set of 3D Gaussian primitives $\mathcal{G} = \{g_i\}$, where each primitive $g_i$ is defined by mean position $\mu_i$, covariance matrix $\Sigma_i$ (or scale/rotation parameters), opacity $\alpha_i$, and spherical harmonics (SH) coefficients $c_{i,lm}$ for color [S1]. These primitives are typically generated via a direct feed-forward process or Structure-from-Motion (SfM) point cloud, potentially pretrained on large datasets [S1].
*   **Output:** A refined set of 3D Gaussians with optimized geometry, opacity, and color attributes that minimize rendering error against input images [S1].

**Data Structures:**
The core data structure is the list of Gaussian primitives. Each primitive $i$ stores:
*   $\mu_i \in \mathbb{R}^3$: Mean position.
*   $\sigma_i \in \mathbb{R}^3$: Scale (log-space).
*   $q_i \in \mathbb{R}^4$: Rotation (quaternion).
*   $\alpha_i \in \mathbb{R}$: Opacity.
*   $c_{i,lm} \in \mathbb{R}$: SH coefficients for color (typically degree 3 or 4) [S1].

**Steps:**
1.  **Initialization:** Predict a set of 3D Gaussian primitives for initialization via a direct feed-forward process. Both geometry (position, scale, rotation) and color attributes (SH coefficients) are generated simultaneously [S1]. This step often leverages pretrained priors to provide a high-quality starting point [S1].
2.  **Optimization Setup:** Initialize the optimizer (e.g., Adam) with a learning rate (typically $1 \times 10^{-3}$) [S1]. Define the loss function, which typically includes photometric loss, SSIM loss, and entropy loss, similar to vanilla 3DGS [S1].
3.  **Joint Optimization:** Optimize all geometry and color attributes directly using gradient descent. This step follows the vanilla 3DGS optimization pipeline, where gradients are backpropagated through the differentiable rasterizer [S1].
4.  **Densification Control:** Enable growing and pruning of 3D primitives during the fine-tuning phase [S1].
    *   **Pruning:** Remove redundant Gaussians that have low opacity or contribute minimally to the scene, significantly reducing the total number of Gaussians and memory consumption [S1].
    *   **Growing:** Split or clone Gaussians in regions with high gradient magnitude or reconstruction error to increase density where needed [S1].
5.  **Convergence:** Iterate until the rendering quality plateaus or a maximum number of steps is reached. The pretrained priors allow this process to converge faster than standard training [S1].

**Formulas:**
The total loss $\mathcal{L}$ is typically a weighted sum:
$$ \mathcal{L} = \lambda_{ssim} \mathcal{L}_{ssim} + \lambda_{e} \mathcal{L}_{entropy} + \mathcal{L}_{photometric} $$
Where $\mathcal{L}_{ssim}$ is the Structural Similarity Index Measure loss, $\mathcal{L}_{entropy}$ encourages opacity to be binary (0 or 1), and $\mathcal{L}_{photometric}$ is the $L_1$ or $L_2$ difference between rendered and ground truth colors [S1].

**Parameters:**
*   Learning Rate: $1 \times 10^{-3}$ [S1].
*   Optimizer: Adam [S1].
*   SH Degree: Typically 3 or 4, depending on the implementation [S1].

**Edge Cases and Pitfalls:**
*   **Redundancy:** The initial feed-forward initialization may produce redundant Gaussians. Without effective pruning, memory consumption remains high [S1].
*   **Overfitting:** Aggressive densification without proper regularization can lead to overfitting on input views, reducing generalization to novel views [S1].
*   **Precision:** Using sparse convolution libraries (e.g., `torchsparse`) requires careful handling of coordinate precision to avoid artifacts in large-scale scenes [S1].

## Subtopics

### [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)
A densification control mechanism for 3D Gaussian Splatting that uses an exponentially ascending gradient threshold to regulate the splitting and cloning of Gaussians during training. It solves the trade-off between early-stage convergence speed and late-stage overfitting by allowing aggressive densification when few Gaussians exist and restricting it as the scene becomes dense. It sets itself apart from static threshold methods by dynamically adjusting the sensitivity of densification decisions based on the normalized training progress. [Exponentially Ascending Gradient Threshold](9684-exponentially-ascending-gradient-threshold.md)

### [Hard Gaussian Splatting](10590-hard-gaussian-splatting.md)
Hard Gaussian Splatting (HGS) is a densification strategy for 3D Gaussian Splatting that identifies "hard Gaussians"—defective primitives causing blurring or needle-like artifacts—and grows them to recover missing scene details. Unlike standard 3DGS, which relies on averaged view-space positional gradients that smooth out local errors and hinder the growth of defective Gaussians, HGS uses multi-view significant positional gradients (PGHGS) and rendering error analysis (REHGS) to detect regions with high reconstruction error. It builds on [Fine-tuning](9138-fine-tuning.md) by refining the Gaussian distribution during the training phase to resolve artifacts that standard adaptive density control misses. [Hard Gaussian Splatting](10590-hard-gaussian-splatting.md)

### [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)
SplatFormer is a learning-based, feed-forward 3D neural module that refines initial 3D Gaussian Splatting (3DGS) representations to enable robust novel view synthesis from out-of-distribution (OOD) camera angles. It addresses the bias of standard 3DGS optimization, which overfits to input views and fails to generalize to unseen high-elevation perspectives, by applying a point transformer network that predicts attribute residuals. Unlike 2D image restoration methods that suffer from multi-view inconsistencies, SplatFormer operates directly on 3D primitives, ensuring geometric consistency while recovering details lost in OOD views. [SplatFormer Training Objective & Refinement](11745-splatformer-training-objective-refinement.md)

### [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)
This method reformulates 3D Gaussian Splatting (3DGS) by treating the set of 3D Gaussians as random samples drawn from an underlying probability distribution describing the scene, specifically as Markov Chain Monte Carlo (MCMC) samples. It converts conventional 3DGS updates into Stochastic Gradient Langevin Dynamics (SGLD) updates by introducing a noise term to promote sample exploration, thereby removing the need for manual densification heuristics like cloning, splitting, and opacity resetting. Unlike sibling methods that rely on complex gradient thresholds or hard constraints, this approach relies on probabilistic state transitions and regularization to manage Gaussian population. [MCMC-based 3DGS / SGLD for 3DGS](11917-mcmc-based-3dgs-sgld-for-3dgs.md)

### [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)
Deblurring 3D Gaussian Splatting is a real-time deblurring framework that reconstructs sharp 3D scenes from blurry input images by manipulating the covariance matrices and means of 3D Gaussians using a small Multi-Layer Perceptron (MLP) during training. Unlike volumetric rendering-based methods like Deblur-NeRF which require seconds per image, this method uses differentiable rasterization to achieve real-time inference speeds (>800 FPS) while handling both defocus and camera motion blur. It addresses the sparse point clouds caused by blurry inputs through K-nearest-neighbor interpolation and depth-based pruning. [Deblurring 3D Gaussian Splatting](12318-deblurring-3d-gaussian-splatting.md)

### [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)
A densification technique designed to compensate for the sparsity of 3D Gaussians at the far end of scenes in defocus-blurry environments, where Structure-from-Motion (SfM) extraction is typically poor. It supplements the initial point cloud with synthetic points to ensure adequate coverage in regions where SfM fails to extract features, thereby improving reconstruction fidelity in large-depth-of-field scenarios. [Supplementary Point Addition for Sparse Point Clouds](12323-supplementary-point-addition-for-sparse-point-clouds.md)

### [Gaussian Volume Mask](12710-gaussian-volume-mask.md)
Gaussian Volume Mask is a densification strategy for 3D Gaussian Splatting that reduces the number of active Gaussians by applying binary masks to scale and opacity attributes based on a learnable parameter. It addresses the excessive memory and storage requirements caused by the proliferation of Gaussians during training, which often include redundant primitives with negligible volume or transparency. Unlike original 3DGS, which relies solely on opacity-based control and stops densifying in the middle of training, this method consistently masks out unessential Gaussians throughout the entire process by considering both geometric volume and transparency. [Gaussian Volume Mask](12710-gaussian-volume-mask.md)

### [Error-Guided Gaussian Reallocation](13012-error-guided-gaussian-reallocation.md)
Error-Guided Gaussian Reallocation (EGR) is a training-stage module that redistributes a fixed budget of Gaussian primitives toward regions with persistent coarse-scale reconstruction errors, without increasing the total number of Gaussians. It refines post-densification Gaussian allocation by moving low-contribution (LC) Gaussians to high-error (HE) regions within a fixed Gaussian set, using contribution- and error-aware selection. Unlike methods using explicit rollback or separate re-correction modules, EGR uses adaptive relocation count, parameter reset, and continued optimization as its stabilization strategy. [Error-Guided Gaussian Reallocation](13012-error-guided-gaussian-reallocation.md)

### [Augmented Radiance Field](13218-augmented-radiance-field.md)
Augmented Radiance Field is a post-enhancement framework that improves 3D Gaussian Splatting (3DGS) scenes by adding Gaussian kernels with view-dependent opacity to explicitly model specular components. It solves the problem that standard spherical harmonics (SH) cannot decouple diffuse and specular radiance, leading to poor reconstruction of sharp highlights and material artifacts. Unlike standard 3DGS which assigns SH coefficients to every primitive, this method uses distinct primitives for diffuse and specular components, avoiding the memory and stability issues of high-order SH. [Augmented Radiance Field](13218-augmented-radiance-field.md)

### ACO-EGR Framework
ACO-EGR is a unified framework that integrates Adaptive-Scale Color Offset (ACO) and Error-Guided Gaussian Reallocation (EGR) into a multi-scale 3DGS backbone to jointly address scale-dependent color mismatch and primitive-allocation imbalance [S2]. The framework augments a multi-scale 3DGS backbone with two training-stage modules: ACO adapts per-Gaussian color across rendering scales through scale-specific DC color offsets, and EGR refines the post-densification Gaussian allocation by moving low-contribution Gaussians to high-error regions within a fixed Gaussian set [S2]. Unlike existing methods that keep fixed color and spatial allocation after densification, ACO-EGR adapts both Gaussian color and primitive allocation together [S2]. ACO-EGR improves rendering quality without modifying the inference-stage rendering process itself; it only changes the training/optimization of the Gaussian parameters [S2].

## Operating characteristics
*   **Time Complexity:** Fine-tuning converges faster than training from scratch due to the quality of the initialization, but the exact speedup depends on the quality of the feed-forward model [S1].
*   **Memory Complexity:** The method significantly reduces memory consumption compared to dense initializations by pruning redundant Gaussians during the fine-tuning phase [S1].
*   **Performance:** Achieves photorealism on par with or surpassing other methods, particularly in urban view synthesis scenarios [S1].
*   **Strengths:** Fast convergence, reduced memory footprint, high fidelity.
*   **Limitations:** Relies on the quality of the initial feed-forward initialization; poor initialization may require more optimization steps or result in suboptimal local minima [S1].
*   **When to Use:** When high-quality initial geometry and color estimates are available (e.g., from pretrained models or SfM) and computational resources are limited.
*   **When Not to Use:** When no good initialization is available, or when the scene has extreme variations that a feed-forward model cannot capture [S1].

## Background
Fine-tuning in the context of 3D Gaussian Splatting was introduced to address the high computational cost and memory requirements of training 3DGS from scratch. The method leverages pretrained weights or feed-forward models to generate an initial set of Gaussian primitives, which are then refined through standard optimization techniques [S1]. This approach was notably demonstrated in EVolSplat, which applied fine-tuning to urban view synthesis, showing that pretrained priors can accelerate network training and enable faster convergence [S1]. The technique builds on the principles of progressive and multi-stage training, where different aspects of the scene are learned at different stages of the optimization process [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
