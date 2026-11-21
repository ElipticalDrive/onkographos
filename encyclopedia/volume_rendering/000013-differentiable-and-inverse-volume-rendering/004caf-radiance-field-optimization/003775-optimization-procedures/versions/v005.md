# Optimization Procedures

Also called: Compute Matching at Intervals, Interval-based Optimal Transport Update, Optimal Transport Matching Interval (K), matching interval

## Overview
Optimization Procedures define the iterative algorithmic framework and specific strategies used to solve inverse rendering problems by minimizing the discrepancy between rendered images and ground-truth observations [G]. It sets itself apart from its siblings by providing the generic control flow—sampling, rendering, loss computation, and parameter update—upon which specific strategies like NeRF training, black-box optimization, or staged refinement are built, rather than defining the specific loss functions or parameter representations themselves [G].

## Shared principles

## Subtopics

### [Color Volume Optimization and Density Initialization](842-color-volume-optimization-and-density-initialization.md)
Addresses the non-convex optimization problem inherent in reconstructing 3D density fields using non-monotonic transfer functions. It sets itself apart by decoupling the problem: first optimizing a "color volume" storing RGB emission and scalar absorption directly to bypass TF-induced local minima, then estimating a density volume from this color volume for robust initialization. Choose this when standard density optimization converges to incorrect local minima due to poor initialization [842-color-volume-optimization-and-density-initialization.md].

### [Regularization losses](1413-regularization-losses.md)
Auxiliary objective functions added to the total training loss to constrain the solution space, prevent overfitting, and enforce physical or structural priors on the scene representation. It sets itself apart from photometric losses by targeting the internal structure or geometry of the scene rather than just final pixel color fidelity. Choose this when standard photometric losses alone cannot resolve undesirable artifacts like floaters or geometric inconsistencies [1413-regularization-losses.md].

### [Optimization Loops and Training Procedures](5622-optimization-loops-and-training-procedures.md)
Defines the iterative optimization procedure that minimizes a photometric or physical loss between rendered images and ground-truth observations by updating scene representation parameters. It sets itself apart from evaluation metrics by using stochastic ray sampling and adaptive integration strategies to facilitate gradient propagation rather than deterministic rendering. Choose this as the foundational training loop for differentiable volume rendering [5622-optimization-loops-and-training-procedures.md].

### [Active Set Method for OIT-based 3DGS](7528-active-set-method-for-oit-based-3dgs.md)
An optimization strategy that accelerates the training of Order-Independent Transparency (OIT) based 3D Gaussian Splatting by exploiting the sparsity of variable dependencies among Gaussian splats. It sets itself apart from standard 3DGS by removing the expensive depth sorting step and freezing converged or negligible Gaussians, optimizing only a subset of "active" variables. Choose this when training speed for OIT-based 3DGS is a bottleneck [7528-active-set-method-for-oit-based-3dgs.md].

### [Photometric Reconstruction Loss](7799-photometric-reconstruction-loss.md)
Minimizes the error between rendered pixel colors and input training images during joint optimization of scene representation and camera parameters. It sets itself apart from standard pixel-wise losses by explicitly mapping 3D sample coordinates from a canonical ray to camera-specific rays via rigid transformations, ensuring geometric consistency across views. Choose this when enforcing geometric consistency across views is critical [7799-photometric-reconstruction-loss.md].

### [Bundle Adjustment](8540-bundle-adjustment.md)
An iterative optimization strategy that refines camera poses (quaternion and translation) for a given view by minimizing the pixel-wise loss between the rendered image and the ground truth image using gradient descent. It sets itself apart by applying per-view optimization loops that refine poses iteratively, often using specific learning rate schedules and masking strategies to handle sparse or noisy renderings. Choose this when initial camera poses are inaccurate and need joint optimization with the scene representation [8540-bundle-adjustment.md].

### [Adversarial and GAN-based Optimization](12094-adversarial-and-gan-based-optimization.md)
Enhances the photorealism and structural consistency of radiance fields by integrating a discriminator network into the training loop. It sets itself apart from pure geometric or photometric approaches by introducing a game-theoretic component that provides gradient feedback for perceptual quality rather than just color accuracy. Choose this when suppressing artifacts and improving high-frequency details is required [12094-adversarial-and-gan-based-optimization.md].

### Compute Matching at Intervals
An optimization strategy to improve speed by computing the pixel matching (transportation matrix T) less frequently than the gradient updates [S1]. Computing pixel matching is costly and is the bottleneck in computation, while there are only subtle changes in pixel matching within adjacent iteration steps [S1]. Instead of computing the matching at each step, it is computed at intervals (i.e., once every $K+1$ steps) for higher speed without hurting accuracy [S1]. The matching interval $K$ controls the frequency of recomputing pixel matching during optimization to accelerate convergence [S1]. During optimization, optimal transport matching computation is skipped for $K-1$ iterations, then recomputed on the $K$-th iteration [S1]. The parameter $K$ is the matching interval; $K=0$ computes matching at every iteration, while $K=5$ is the default, providing a trade-off between speed and quality [S1]. Setting $K=5$ or $K=10$ generally leads to similar result qualities compared to $K=0$, while increasing the running speed by $4\times$ to $5\times$ [S1].

### Hybrid Gradients
A hybrid optimizing strategy that combines RGBXY derivatives and standard derivatives, leveraging the advantages of both [S1]. RGBXY derivatives are highly effective in capturing global and long-range influence, but may be less effective when initial and target shapes are already aligned very well, potentially due to inaccurate matching from the approximated optimal transport method when shapes are close [S1]. The hybrid strategy uses RGBXY derivatives for the first 75% of iterations and switches to standard derivatives for the last 25% of iterations [S1]. A randomized strategy for combining them (randomly selecting one at each iteration) does not show benefits [S1].

### Per-Scene Fine-tuning
A post-processing step where the predicted voxel features of the global radiance field are optimized per scene to boost rendering quality [S2]. The process takes the global radiance field $V_g$ predicted by the network and optimizes the predicted voxel features per scene using rendering losses for a short period [S2]. Fine-tuning takes approximately 20 minutes to 1 hour, resulting in state-of-the-art quality that outperforms NeRF [S2].

### Two-Stage Optimization with Edge-Aware Surface Rendering
A staged optimization procedure for inverse rendering [G]. Stage 1 trains a volumetric radiance field for 100k iterations, sampling 512 random pixels per iteration, and optimizes using L1 image loss and eikonal regularization loss with weight $\lambda_1 = 0.1$ [S3]. Stage 2 trains an edge-aware physics-based surface rendering for 50k iterations, rendering a random $128 \times 128$ image patch per iteration, and optimizes using L2 image loss over 4 Gaussian pyramid levels, eikonal loss ($\lambda_1 = 0.1$), and roughness range loss ($\lambda_2 = 0.1$) [S3]. The depth gradient magnitude threshold for edge localization is $\tau = 10^{-2}$, the maximum number of surface walk steps is $K = 16$, the step size for surface walk is $\epsilon = 10^{-3}$, and the dot-product threshold for edge point localization is $\delta = 5 \times 10^{-2}$ [S3]. The BRDF distribution parameter is 'ggx', with intIOR, extIOR, and nonlinear parameters using Mitsuba roughplastic defaults [S3]. Total training time for both stages is ~10 hours on a single NVIDIA RTX 2080 Ti GPU with 12GB memory [S3].

## Background
The optimization procedures described herein originate from various specific research papers addressing different aspects of radiance field reconstruction [G]. The interval-based optimal transport update and hybrid gradients are detailed in "Differentiable Rendering Using RGBXY Derivatives and Optimal Transport" [S1]. The per-scene fine-tuning approach is introduced in "NeRFusion: Fusing Radiance Fields for Large-Scale Scene Reconstruction" [S2]. The two-stage optimization with edge-aware surface rendering is presented in "IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images" [S3].

## Sources
[S1] Differentiable Rendering Using RGBXY Derivatives and Optimal Transport, https://doi.org/10.1145/3550454.3555479, fetched 2026-10-27
[S2] NeRFusion: Fusing Radiance Fields for Large-Scale Scene Reconstruction, https://doi.org/10.1109/cvpr52688.2022.00537, fetched 2026-10-26
[S3] IRON: Inverse Rendering by Optimizing Neural SDFs and Materials from Photometric Images, https://doi.org/10.1109/cvpr52688.2022.00548, fetched 2026-10-26
