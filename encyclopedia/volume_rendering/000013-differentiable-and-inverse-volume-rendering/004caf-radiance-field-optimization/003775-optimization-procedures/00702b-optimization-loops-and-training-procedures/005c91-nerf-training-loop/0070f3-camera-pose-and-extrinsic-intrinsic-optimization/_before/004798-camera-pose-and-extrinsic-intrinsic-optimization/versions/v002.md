# Camera Pose and Extrinsic/Intrinsic Optimization

Also called: Camera Pose and Parameter Optimization, Camera Preconditioning, Time-dependent NeRF with Continuous Camera Motion

## Overview
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP [S1]. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets [S1]. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence.

## Shared principles
All methods in this family share the core mechanism of differentiable rendering, where the rendered pixel color $\hat{C}(r)$ is a differentiable function of the camera pose $P$ and the NeRF parameters $\Theta$. The optimization problem is typically formulated as minimizing a loss function $\mathcal{L}$ with respect to both $\Theta$ and $P$:
$$ \min_{\Theta, P} \mathcal{L}(\hat{C}(r; \Theta, P), C(r)) $$
where $C(r)$ is the ground-truth pixel color. The gradient $\nabla_P \mathcal{L}$ allows the camera pose to be updated via an optimizer (e.g., Adam, SGD) in lockstep with the NeRF weights. Key shared components include:
1.  **Ray Casting:** Computing ray origins $\hat{o}$ and directions $\hat{d}$ from pixel coordinates using current camera estimates. For a pinhole camera model with focal length $f$ and rotation $R$ and translation $t$:
    $$ \hat{o} = t $$
    $$ \hat{d} = R \begin{pmatrix} (u - W/2)/f \\ -(v - H/2)/f \\ -1 \end{pmatrix} $$
2.  **Differentiable Rendering:** The volume rendering integral is computed via discrete summation, allowing backpropagation through the ray marching process to the pose parameters [S1].
3.  **Joint Optimization:** Separate optimizers or a combined optimizer are used to update NeRF weights and pose parameters, often with different learning rates to stabilize convergence [S1].

## Subtopics

### [Camera Pose and Parameter Optimization](7611-camera-pose-and-parameter-optimization.md)
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence.

### [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md)
This method jointly optimizes a time-dependent Neural Radiance Field (NeRF) and continuous camera motion from monocular video, representing the scene as local Signed Distance Field (SDF) and appearance fields at each time step. Unlike methods that train a single global NeRF mapped via noisy initial transformations, this approach uses local time-dependent representations constrained by consistency losses to avoid local minima. It builds on [Camera Pose and Extrinsic/Intrinsic Optimization](12051-camera-pose-and-extrinsic-intrinsic-optimization.md) by modeling camera motion as continuous velocities rather than discrete pose parameters.

### [Camera Preconditioning](12545-camera-preconditioning.md)
Camera Preconditioning (CamP) is a technique that normalizes and decorrelates camera parameters during Neural Radiance Field (NeRF) optimization by transforming the optimization problem from direct camera parameters to latent preconditioned parameters. It addresses the ill-conditioning caused by mixed units (e.g., translation in world units vs. focal length in pixels) and spurious correlations between parameters, which cause naive optimization to converge slowly or inaccurately. Unlike per-parameter scaling or adaptive optimizers like Adam, which may ignore sensitivity differences or fail to decouple correlated coefficients, CamP uses a preconditioner derived from the covariance of projection derivatives to ensure uniform sensitivity across all parameter dimensions.

### RoDynRF
- [S1] Enables dynamic scene reconstruction from monocular videos without known camera poses by jointly optimizing intrinsics, extrinsics, and both static and dynamic radiance fields.
- [S1] Introduce a two-branch architecture separating background and moving regions.
- [S1] Use motion masks to exclude dynamic pixels during pose estimation.
- [S1] Employ a coarse-to-fine voxel optimization scheme for stability.
- [S1] Use a time-conditioned deformation network to model non-rigid motion.

### NeRF Pose Optimization
- [S1] Formulates pose reconstruction as an inverse problem by optimizing the pose instead of network parameters given a pre-trained NeRF.
- [S1] Use the photo-metric loss Eq. 6 to optimize the pose.
- [S1] Use an interest-point detector and perform interest region-based sampling.

### Bundle-Adjusted Neural Radiance Field (BARF)
- [S1] Jointly estimates poses alongside the training of the neural radiance field using coarse-to-fine registration by adaptively masking the positional encoding.
- [S1] Use coarse-to-fine registration by adaptively masking the positional encoding.
- [S1] Use naive dense ray sampling for simplicity.

### NeRF Self-Calibration and Pose Optimization
- [S1] Self-calibrating joint optimization model for NeRF that optimizes unknown poses and camera intrinsic parameters for non-linear camera models such as fish-eye lens models.
- [S1] Use curriculum learning to gradually introduce the nonlinear camera/noise parameters to the joint optimization.

## Background
The problem of camera pose estimation is traditionally solved via Structure-from-Motion (SfM) and Bundle Adjustment (BA) [S1]. However, early NeRF implementations relied heavily on pre-computed poses from SfM pipelines like COLMAP, which can fail on sparse or uncalibrated image sets [S1]. iNeRF (December 2020) [S1] formulated pose reconstruction as an inverse problem, optimizing the pose instead of network parameters given a pre-trained NeRF using photo-metric loss. Concurrently, NeRF– (February 2021) [S1] jointly estimated NeRF model parameters and camera parameters, achieving comparable results to COLMAP but struggling with rotational motion [S1]. Bundle-Adjusted Neural Radiance Field (BARF) (April 2021) [S1] improved upon this using coarse-to-fine registration by adaptively masking positional encoding, outperforming NeRF– on LLFF scenes [S1]. Jeong et al. introduced SCNeRF (August 2021) [S1], which self-calibrates joint optimization for non-linear camera models like fish-eye lenses using curriculum learning, outperforming BARF [S1]. GNeRF (March 2021) [S1] used pose as a generative latent code with adversarial training [S1]. More recently, RoDynRF (January 2023) [S1] enabled dynamic scene reconstruction from monocular videos without known poses by jointly optimizing intrinsics, extrinsics, and radiance fields [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
