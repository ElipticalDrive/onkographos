# Camera Pose and Parameter Optimization

Also called: Joint Pose and NeRF Optimization, Differentiable Camera Optimization

## Overview
This family of methods addresses the problem of optimizing camera parameters (extrinsics and intrinsics) simultaneously with Neural Radiance Field (NeRF) weights, eliminating the reliance on pre-computed Structure-from-Motion (SfM) pipelines like COLMAP [S2, unofficial]. By treating camera poses as learnable parameters, these methods enable 3D reconstruction from sparse or uncalibrated image sets [S2, unofficial]. What sets this family apart from siblings like [Time-dependent NeRF with Continuous Camera Motion](12399-time-dependent-nerf-with-continuous-camera-motion.md) is the explicit optimization of discrete camera poses per image rather than continuous motion functions, and from [Camera Preconditioning](12545-camera-preconditioning.md) by focusing on the joint gradient-based update of poses and scene geometry rather than preconditioning the optimizer for faster convergence.

## Shared principles
All methods in this family share the core mechanism of differentiable rendering, where the rendered pixel color $\hat{C}(r)$ is a differentiable function of the camera pose $P$ and the NeRF parameters $\Theta$. The optimization problem is typically formulated as minimizing a loss function $\mathcal{L}$ with respect to both $\Theta$ and $P$:
$$ \min_{\Theta, P} \mathcal{L}(\hat{C}(r; \Theta, P), C(r)) $$
where $C(r)$ is the ground-truth pixel color. The gradient $\nabla_P \mathcal{L}$ allows the camera pose to be updated via an optimizer (e.g., Adam, SGD) in lockstep with the NeRF weights. Key shared components include:
1.  **Ray Casting:** Computing ray origins $\hat{o}$ and directions $\hat{d}$ from pixel coordinates using current camera estimates. For a pinhole camera model with focal length $f$ and rotation $R$ and translation $t$:
    $$ \hat{o} = t $$
    $$ \hat{d} = R \begin{pmatrix} (u - W/2)/f \\ -(v - H/2)/f \\ -1 \end{pmatrix} $$
2.  **Differentiable Rendering:** The volume rendering integral is computed via discrete summation, allowing backpropagation through the ray marching process to the pose parameters [S2, unofficial].
3.  **Joint Optimization:** Separate optimizers or a combined optimizer are used to update NeRF weights and pose parameters, often with different learning rates to stabilize convergence [S2, unofficial].

## Subtopics

### [NeRF with Unknown Camera Parameters](7405-nerf-with-unknown-camera-parameters.md)
Simultaneously optimizes camera poses and NeRF parameters without pre-estimated intrinsics/extrinsics. It solves reconstruction from sparse, forward-facing views where camera parameters are unknown. Unlike baseline NeRF, it treats poses as learnable parameters, achieving NVS quality comparable to baseline NeRF ($\Delta$SSIM and $\Delta$LPIPS $\approx 0.05$) without external pose estimation.

### [Hybrid 2D/3D Registration Optimization Pipeline](8120-hybrid-2d-3d-registration-optimization-pipeline.md)
A two-stage method for 2D X-ray to 3D CT registration. It combines a differentiable neural network similarity loss for global convergence with a Gradient-based Normalized Cross-Correlation (Grad-NCC) loss for local refinement. It leverages network features for coarse alignment and gradient-based correlation for fine-tuning, decoupling the optimization landscape into two phases.

### [NoPe-NeRF](8705-nope-nerf.md)
Trains NeRFs without precomputed poses by optimizing relative poses between consecutive frames using undistorted monocular depth priors. It differs from global alignment methods by generating a dense point cloud from monocular depth and constraining optimization via point cloud and photometric losses.

### [AltNeRF](8763-altnerf.md)
Creates resilient NeRFs from monocular videos without known poses using self-supervised monocular depth estimation (SMDE) for depth and pose priors. It uses an alternating algorithm to harmonize NeRF outputs with SMDE through error-tolerant regularization, progressively refining geometry and camera parameters.

### [Vision-Only Pose Filtering in a NeRF](10501-vision-only-pose-filtering-in-a-nerf.md)
An optimization-based state estimation filter for robot pose estimation within a NeRF using only monocular RGB images. It treats the problem as recursive Bayesian estimation, balancing NeRF-based photometric loss with dynamics prediction loss, and explicitly incorporates system dynamics and uncertainty propagation.

### [Online Receding Horizon Replanning](10509-online-receding-horizon-replanning.md)
A vision-based robot navigation pipeline combining trajectory planning and pose filtering in a receding horizon loop. It solves navigation with uncertain initial poses by continuously replanning trajectories based on updated state beliefs, allowing changes in homotopy classes around obstacles.

### [NeRF-based Trajectory Optimization](10632-nerf-based-trajectory-optimization.md)
Optimizes dense flat output waypoints for differentially flat systems (e.g., quadrotors) using first-order gradient descent and NeRF density for implicit obstacle representation. It differs from static waypoint planners by using optimizable waypoints and implicit NeRF obstacles, requiring nonconvex optimization via automatic differentiation.

### NeRF Registration
#### definition
- [S1] A rigid NeRF registration method that minimizes a photometric loss between rendered images from two NeRFs, using VF and Normalizing Flows to guide ray sampling and generate novel views.
#### step
- [S1] Optimize a 6-DoF rigid transformation parameter (rotation and translation) to align the two NeRFs.
- [S1] Minimize a photometric loss (Equation 7 in the full paper) between rendered images from the aligned NeRFs.
- [S1] Use VF and Normalizing Flows to sample novel camera viewpoints and guide ray sampling during optimization.
#### parameter
- [S1] learnable scale factor (added to the 6-DoF parameters to handle unknown scale and illumination changes between scenes).
#### pitfall
- [S1] Photometric loss is susceptible to inaccuracies on textureless surfaces.
- [S1] High symmetry can cause convergence to a partially accurate solution that deviates by 180° from the true solution.
- [S1] Partial overlap or moving objects are not explicitly handled and may cause failure.

### Progressive Camera Pose Generation
#### definition
- [S2, unofficial] A training strategy that gradually increases the angular distance of sampled target viewpoints from source views to stabilize optimization.
#### step
- [S2, unofficial] Sample noise uniformly from interval $[-\beta, +\beta]$ and add it to original Euler rotation angles of input view poses.
#### parameter
- [S2, unofficial] Parameter $\beta$: Grows linearly from 3 to 9 degrees throughout the course of optimization.

## Background
The problem of optimizing camera poses jointly with NeRF parameters was first explicitly addressed in the context of unknown camera parameters by [NeRF with Unknown Camera Parameters](7405-nerf-with-unknown-camera-parameters.md), which demonstrated that joint optimization could achieve quality comparable to COLMAP-initialized NeRFs [S2, unofficial]. Subsequent work expanded this to include depth priors ([NoPe-NeRF](8705-nope-nerf.md), [AltNeRF](8763-altnerf.md)) and specific applications like medical imaging ([Hybrid 2D/3D Registration](8120-hybrid-2d-3d-registration-optimization-pipeline.md)) and robotics ([Vision-Only Pose Filtering](10501-vision-only-pose-filtering-in-a-nerf.md), [Online Receding Horizon Replanning](10509-online-receding-horizon-replanning.md), [NeRF-based Trajectory Optimization](10632-nerf-based-trajectory-optimization.md)). The technique of NeRF registration using Viewshed Fields was introduced in [VF-NeRF](S1).

## Sources
[S1] VF-NeRF: Viewshed Fields for Rigid NeRF Registration, https://doi.org/10.1007/978-3-031-73027-6_10, fetched 2026-10-26
[S2] GeCoNeRF: Few-Shot Neural Radiance Fields via Geometric Consistency, https://core.ac.uk/works/309257742, fetched 2026-11-01 (unofficial)
