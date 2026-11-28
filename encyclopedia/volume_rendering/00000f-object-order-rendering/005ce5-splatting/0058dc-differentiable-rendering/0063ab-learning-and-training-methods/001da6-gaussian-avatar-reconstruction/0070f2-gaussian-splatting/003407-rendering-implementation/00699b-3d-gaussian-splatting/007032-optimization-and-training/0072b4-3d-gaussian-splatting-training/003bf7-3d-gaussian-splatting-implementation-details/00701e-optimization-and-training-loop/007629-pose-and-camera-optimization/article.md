# Pose and Camera Optimization

Also called: Camera Pose Optimization, View Pose Estimation

## Overview
Pose and Camera Optimization refers to the family of methods that estimate the relative or absolute camera poses (rigid transformations) by minimizing the photometric error between rendered images and ground-truth views using a 3D Gaussian Splatting representation. It solves the problem of recovering camera motion without relying on external Structure-from-Motion (SfM) pipelines, enabling incremental or global pose estimation directly from differentiable rendering. It sets itself apart from sibling methods like [3D Gaussian Splatting Training Loop](15139-3d-gaussian-splatting-training-loop.md) by treating camera poses as learnable parameters rather than fixed inputs, and distinguishes itself from [Local 3DGS Optimization](20073-local-3dgs-optimization.md) and [Neural Parametric Gaussians Optimization](21236-neural-parametric-gaussians-optimization.md) by encompassing both local incremental estimation and global non-rigid deformation contexts.

## Shared principles
The core mechanism common to all methods in this family is the minimization of a photometric loss $\mathcal{L}$ between the rendered image $\hat{I}$ and the ground-truth image $I_{gt}$ [S1]. The standard photometric baseline is a weighted sum of L1 loss and differentiable Structural Similarity (D-SSIM) [S1]:
$$ \mathcal{L}_{img} = (1 - \lambda) \mathcal{L}_{L1} + \lambda \mathcal{L}_{D-SSIM} $$
where $\mathcal{L}_{L1}$ is the average absolute difference between corresponding pixel colors, and $\mathcal{L}_{D-SSIM}$ measures local structure, contrast, and texture . The weight $\lambda$ is typically 0.2 [S1].

The optimization uses the Adam optimizer with $\beta_1 = 0.9$ and $\beta_2 = 0.99$ [S1]. Training typically runs for a fixed number of iterations, commonly around 30,000 for standard scenes, though specific schedules vary (e.g., [Gaussian Primitive Training Schedule](18977-gaussian-primitive-training-schedule.md) uses ~6,000) . The process does not stop at a specific loss threshold because the loss curve continues to creep down with diminishing returns; a fixed iteration budget is used instead .

Gaussian parameters $\mathcal{G} = \{g_i\}_{i=1}^N$ include position $\mathbf{p}_i \in \mathbb{R}^3$, covariance $\Sigma_i$ (parameterized by rotation quaternion $\mathbf{q}_i \in \mathbb{R}^4$ and scale vector $\mathbf{s}_i \in \mathbb{R}^3$), opacity $\alpha_i \in [0, 1]$, and Spherical Harmonics (SH) coefficients $\mathbf{c}_{i, l}$ for view-dependent color [S1].

## Subtopics

### [Local 3DGS Optimization](20073-local-3dgs-optimization.md)
Local 3DGS Optimization is an iterative method for estimating the relative camera pose (transformation) between two nearby images by optimizing a local 3D Gaussian Splatting representation against photometric loss. It solves the problem of camera pose estimation without requiring external Structure-from-Motion (SfM) pipelines like COLMAP, relying instead on monocular depth estimation and differentiable rendering. It sets itself apart from global optimization methods by freezing the Gaussian attributes (position, scale, rotation, opacity, SH) after initialization and optimizing only the pose parameters, making it suitable for incremental or local sequence processing [20073-local-3dgs-optimization.md].

### [Neural Parametric Gaussians Optimization](21236-neural-parametric-gaussians-optimization.md)
Neural Parametric Gaussians (NPGs) is a two-stage optimization procedure for monocular non-rigid object reconstruction that combines a coarse parametric point model regularized by a low-rank deformation basis with 3D Gaussians to model fine geometric and appearance details efficiently. It sets itself apart from siblings like [Local 3DGS Optimization](20073-local-3dgs-optimization.md) by explicitly decoupling the learning of global non-rigid deformation (via the parametric basis) from local detail fitting (via Gaussians), ensuring coarse geometry remains intact over time.

### Global Latent Optimization (GLO)
#### definition
- [S1] A method to handle photometric inconsistencies (automatic exposure/white-balance) by co-optimizing a per-view latent vector mapped via an MLP to a linear transformation of spherical harmonic coefficients.
#### formula
- [S1] $\hat{c}_i = a \odot c_i + b$, where $\hat{c}_i$ are the transformed coefficients, $c_i$ are the original spherical harmonic coefficients, $\odot$ is the Hadamard product, $b$ models brightness changes, and $a$ provides white-balance expressive power. $a, b = Q(z_n; \theta_Q)$, where $Q$ is an MLP taking latent $z_n \in \mathbb{R}^{64}$.
#### parameter
- [S1] $z_n \in \mathbb{R}^{64}$, a latent vector per input camera view.
- [S1] $\theta_Q$, trainable parameters of the MLP $Q$ that maps $z_n$ to $a$ and $b$.
#### pitfall
- [S1] A reduced model prevents $z_n$ from explaining distractors as per-image adjustments, which would happen in a simpler GLO.

## Background
The concept of optimizing camera poses within differentiable rendering frameworks has evolved from traditional Structure-from-Motion (SfM) pipelines to end-to-end differentiable approaches. Early works like [Martin-Brualla et al. 2021b] (referenced in [S1]) proposed Global Latent Optimization (GLO) to handle per-image adjustments, but suffered from explaining distractors as photometric changes. [Rematas et al. 2022] (referenced in [S1]) provided an analysis of such models. The adaptation of these concepts to 3D Gaussian Splatting, as seen in [SpotLessSplats] [S1], allows for robust pose estimation and photometric consistency handling in casual captures with automatic exposure and white-balance. Local methods, such as those described in [20073-local-3dgs-optimization.md], leverage monocular depth estimation (e.g., DPT) to initialize local Gaussians, enabling pose estimation without external SfM. Non-rigid extensions, like Neural Parametric Gaussians [S1], extend this to dynamic scenes by coupling parametric deformation bases with Gaussian splats.

## Sources
[S1] SpotLessSplats: Ignoring Distractors in 3D Gaussian Splatting, https://doi.org/10.1145/3727143, fetched 2026-10-27
