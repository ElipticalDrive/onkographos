# Fine-tuning
Also called: 3DGS Fine-tuning, Gaussian Splatting Fine-tuning

## Overview
Fine-tuning in 3D Gaussian Splatting (3DGS) refers to the phase where a pre-initialized set of Gaussian primitives is optimized to achieve high-fidelity novel view synthesis, often leveraging pretrained priors or feed-forward initialization to accelerate convergence and reduce memory overhead [S1]. It distinguishes itself from standard training by decoupling the initial geometry/color prediction from the subsequent differentiable optimization, allowing for aggressive densification (growing and pruning) during the refinement stage to correct redundancies inherent in the initialization [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of initial 3D Gaussian primitives $\{G_i\}$ generated via a direct feed-forward process (e.g., from a pretrained network or sparse SfM point cloud), where each primitive $i$ has geometry (mean $\mu_i$, covariance $\Sigma_i$) and color attributes (spherical harmonics coefficients $c_i$) [S1].
*   **Output:** Optimized Gaussian primitives with refined geometry and color attributes, resulting in a scene representation with reduced redundancy and higher photorealism [S1].

**Data Structures:**
The data structure remains the standard 3DGS primitive list. Each Gaussian $i$ stores:
*   Mean position $\mu_i \in \mathbb{R}^3$
*   Covariance matrix $\Sigma_i$ (parameterized as rotation $R_i \in SO(3)$ and scale $s_i \in \mathbb{R}^3$)
*   Opacity $\alpha_i \in [0, 1]$
*   Spherical Harmonics coefficients $c_i$ (typically up to degree 3)

**Steps:**
1.  **Initialization:** Predict a set of 3D Gaussian primitives via a direct feed-forward process. Both geometry and color attributes are generated simultaneously [S1].
2.  **Optimization Setup:** Initialize the optimizer (e.g., Adam) with a learning rate (typically $1 \times 10^{-3}$) [S1].
3.  **Differentiable Rendering:** Render images from training views using differentiable Gaussian rasterization [S1].
4.  **Loss Computation:** Compute the total loss $\mathcal{L}$, typically combining photometric loss, SSIM, and entropy regularization:
    $$ \mathcal{L} = \mathcal{L}_{\text{photometric}} + \lambda_{\text{ssim}} \mathcal{L}_{\text{ssim}} + \lambda_e \mathcal{L}_{\text{entropy}} $$
    where $\lambda_{\text{ssim}}$ and $\lambda_e$ are weighting factors [S1].
5.  **Backpropagation and Update:** Compute gradients with respect to all geometry and color attributes and update parameters directly, following the vanilla 3DGS optimization loop [S1].
6.  **Densification Control:** Enable growing (splitting/cloning) and pruning of 3D primitives during this phase. This step is critical to remove redundant primitives introduced during initialization and to adapt density to scene complexity [S1].
7.  **Iteration:** Repeat steps 3–6 until convergence. The number of Gaussians is significantly reduced compared to the initial feed-forward estimate, maintaining high fidelity while lowering memory consumption [S1].

**Parameters:**
*   **Learning Rate:** $1 \times 10^{-3}$ (Adam optimizer) [S1].
*   **Optimization Library:** `gsplat` for rasterization, `torchsparse` for sparse convolution if volume-based priors are used [S1].

**Edge Cases and Pitfalls:**
*   **Redundancy:** The initial feed-forward step may generate redundant Gaussians. Failure to prune during fine-tuning leads to excessive memory usage and potential overfitting [S1].
*   **Convergence Speed:** While fine-tuning accelerates convergence compared to training from scratch, poor initialization can lead to local minima. Pretrained priors help mitigate this [S1].

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
ACO-EGR is a unified framework that integrates ACO and EGR into a multi-scale 3DGS backbone to jointly address scale-dependent color mismatch and primitive-allocation imbalance [S2]. The framework augments a multi-scale 3DGS backbone with two training-stage modules: ACO (Adaptive-Scale Color Offset) and EGR (Error-Guided Gaussian Reallocation) [S2]. Unlike existing methods that keep fixed color and spatial allocation after densification, ACO-EGR adapts both Gaussian color and primitive allocation together [S2]. ACO-EGR improves rendering quality without modifying the inference-stage rendering process itself; it only changes the training/optimization of the Gaussian parameters [S2].

## Operating characteristics
*   **Time Complexity:** Fine-tuning typically converges faster than training from scratch due to the quality of the initialization, reducing the total number of optimization steps required [S1].
*   **Memory Complexity:** By pruning redundant primitives during the fine-tuning phase, the final memory footprint is significantly lower than the initial feed-forward estimate, though it may still be higher than a manually tuned sparse representation [S1].
*   **Performance:** Achieves photorealism on par with or surpassing other methods, particularly in urban view synthesis scenarios [S1].
*   **Strengths:** Accelerated convergence, reduced memory consumption through pruning, and high fidelity [S1].
*   **Limitations:** Relies on the quality of the initial feed-forward prediction; poor initialization may require more fine-tuning steps or result in suboptimal local minima [S1].

## Background
The concept of fine-tuning 3D Gaussian Splatting emerged as a strategy to leverage pretrained models or efficient initialization schemes to overcome the slow convergence and high memory costs of standard 3DGS training. EVolSplat introduced a method where a feed-forward process predicts initial Gaussians, which are then fine-tuned using standard 3DGS optimization, enabling efficient volume-based Gaussian splatting for urban view synthesis [S1]. This approach highlights the benefit of decoupling initialization from optimization to manage the trade-off between reconstruction quality and computational resources [S1].

## Sources
[S1] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S2] Adaptive-Scale Color Offset and Error-Guided Gaussian Reallocation for Multi-Scale 3D Gaussian Splatting, https://doi.org/10.3390/math14152713, fetched 2026-11-01
