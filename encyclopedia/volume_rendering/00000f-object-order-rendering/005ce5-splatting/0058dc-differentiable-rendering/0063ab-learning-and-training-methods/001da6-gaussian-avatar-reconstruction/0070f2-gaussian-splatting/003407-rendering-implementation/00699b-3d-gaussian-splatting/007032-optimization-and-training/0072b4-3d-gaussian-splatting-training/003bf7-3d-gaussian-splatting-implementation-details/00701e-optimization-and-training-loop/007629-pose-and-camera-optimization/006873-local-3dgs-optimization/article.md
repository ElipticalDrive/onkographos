# Local 3DGS Optimization
Also called: Local 3DGS Pose Optimization

## Overview
Local 3DGS Optimization is an iterative method for estimating the relative camera pose (extrinsic transformation) between two nearby images by optimizing a local 3D Gaussian Splatting (3DGS) representation against photometric loss [S1]. Unlike global reconstruction methods that optimize scene geometry and camera poses simultaneously, this method keeps the local Gaussian attributes (position, scale, rotation, opacity, and spherical harmonics) fixed after initialization, learning only the pose parameters [S1]. It is distinct from [Neural Parametric Gaussians Optimization](21236-neural-parametric-gaussians-optimization.md) by relying on explicit 3D Gaussian primitives rather than neural field representations for the local scene model.

## How it works
**Inputs:**
- Two nearby images: current view $I_t$ and target view $I_{t+1}$ [S1].
- Monocular depth estimation model (e.g., DPT) [S1].
- Camera intrinsic parameters (or heuristics if unknown) [S1].

**Outputs:**
- Estimated relative pose $Q_t$ representing the transformation from the coordinate system of $I_t$ to $I_{t+1}$ [S1].

**Data Structures:**
- **Local 3DGS Representation ($G_t$):** A set of $N$ Gaussians. Each Gaussian $g_i$ contains:
  - Position $\mathbf{p}_i \in \mathbb{R}^3$ [S1].
  - Covariance $\Sigma_i$, parameterized by rotation quaternion $\mathbf{q}_i \in \mathbb{R}^4$ and scale vector $\mathbf{s}_i \in \mathbb{R}^3$ [S1].
  - Opacity $\alpha_i \in [0, 1]$ [S1].
  - Spherical Harmonics (SH) coefficients $\mathbf{c}_{i, l}$ for view-dependent color [S1].
- **Pose Parameter ($T_t$):** A learnable transformation matrix (or quaternion/translation pair) representing the relative pose [S1]. Initialized as Identity $I$ [S1].

**Algorithm Steps:**

1.  **Initialize Local 3DGS:**
    - Compute monocular depth map $D_t$ for image $I_t$ using the depth estimation model (e.g., DPT) [S1].
    - Initialize the local 3DGS representation $G_t$ from $I_t$ and $D_t$ using `InitGauss($I_t, D_t$)` [S1].
    - *Note:* The source mentions downsampling the point cloud before fitting to handle high-resolution inputs [S1].
    - Optimize $G_t$ on the input view $I_t$ via photometric loss for 500 iterations to fit the local geometry [S1].

2.  **Initialize Pose:**
    - Set the relative pose parameter $T_t$ to the Identity transformation $I$ [S1].

3.  **Optimize Pose:**
    - While not converged:
        - Render the target view image $\hat{I}_{t+1}$ by rasterizing the transformed Gaussians $T_t \odot G_t$ [S1].
        - Compute the photometric loss $\mathcal{L}$ between the target image $I_{t+1}$ and the rendered image $\hat{I}_{t+1}$ [S1].
        - Update the pose parameter $T_t$ using the Adam optimizer on the gradient $\nabla \mathcal{L}$ [S1].
        - *Constraint:* All attributes of $G_t$ (position, SH, opacity, scale, rotation) are frozen during this step; only the pose parameters (quaternion and translation vector) are updated [S1].
    - The pose optimization runs for 300 steps [S1].

4.  **Output Pose:**
    - The final relative pose $Q_t$ is the accumulated transformation $T_t$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def local_3dgs_optimization(I_t, I_t_plus_1, depth_model, camera_intrinsics):
    """
    Estimates relative pose between I_t and I_t_plus_1 using Local 3DGS.
    
    Args:
        I_t: Current view image.
        I_t_plus_1: Target view image.
        depth_model: Monocular depth estimation model (e.g., DPT).
        camera_intrinsics: Camera intrinsic parameters.
        
    Returns:
        T_t: Estimated relative pose transformation.
    """
    # Step 1: Initialize Local 3DGS
    # Compute depth for current view
    D_t = depth_model.predict(I_t)
    
    # Initialize Gaussians from image and depth
    # Note: Source mentions downsampling point cloud before fitting [S1]
    G_t = InitGauss(I_t, D_t)
    
    # Optimize local 3DGS attributes on I_t for 500 iterations
    # Loss is photometric loss between rendered I_t and I_t
    for _ in range(500):
        I_rendered = rasterize(G_t, camera_intrinsics)
        L = photometric_loss(I_t, I_rendered)
        # Update G_t attributes (position, scale, rotation, opacity, SH)
        G_t = adam_update(G_t, L)
        
    # Step 2: Initialize Pose
    # T_t represents the transformation from I_t coordinate system to I_t_plus_1
    T_t = Identity()
    
    # Step 3: Optimize Pose
    # Freeze G_t attributes, only optimize T_t
    for _ in range(300):
        # Transform Gaussians by current pose estimate
        G_transformed = transform(G_t, T_t)
        
        # Render target view
        I_rendered_target = rasterize(G_transformed, camera_intrinsics)
        
        # Compute loss against target image
        L = photometric_loss(I_t_plus_1, I_rendered_target)
        
        # Update only pose parameters (quaternion and translation)
        T_t = adam_update(T_t, L)
        
    return T_t

def photometric_loss(I_gt, I_pred):
    """
    Computes weighted sum of L1 loss and D-SSIM.
    Lambda = 0.2
    """
    lambda_weight = 0.2
    l1_loss = mean_absolute_error(I_gt, I_pred)
    ssim_loss = 1.0 - ssim(I_gt, I_pred)
    return (1 - lambda_weight) * l1_loss + lambda_weight * ssim_loss
```

**Parameters:**
- **Local 3DGS Optimization Iterations:** 500 iterations [S1].
- **Pose Learning Optimization Steps:** 300 steps [S1].
- **Monocular Depth Estimation Model:** DPT (Vision Transformers for Dense Prediction) [S1].
- **Camera Intrinsic Heuristic:** If intrinsic parameters are unknown, Field of View (FoV) is set to $79^\circ$ and principal points are set to the image center [S1].
- **Optimizer:** Adam with $\beta_1 = 0.9$ and $\beta_2 = 0.99$ [S1].

**Edge Cases and Pitfalls:**
- **Unknown Intrinsics:** Using heuristic intrinsics (FoV $79^\circ$, center principal point) leads to slight degradation in novel view synthesis and camera pose estimation accuracy compared to known intrinsics [S1].
- **High-Resolution Inputs:** High-resolution images can generate a huge amount of points; downsampling the point cloud before fitting is necessary to manage memory and computation [S1].
- **Dramatic Camera Motion:** The method is designed to handle dramatic camera motions (large relative rotation angles, up to $180^\circ$ in tested sequences like CO3D-V2 and Tanks and Temples) [S1].

## Operating characteristics
- **Time Complexity:** The method involves two optimization phases: 500 iterations for local 3DGS fitting and 300 steps for pose optimization. The complexity is dominated by the rasterization and loss computation per iteration.
- **Memory Complexity:** Depends on the number of Gaussians $N$ in the local representation. Downsampling is used to mitigate memory issues with high-resolution inputs [S1].
- **Strengths:** Capable of handling dramatic camera motions (large relative rotation angles) without requiring COLMAP or initial pose estimates [S1]. It provides photo-realistic rendering with detailed geometry [S1].
- **Limitations:** Performance degrades slightly when camera intrinsics are unknown and must be heuristically set [S1]. It is limited to "nearby" images as defined by the local window assumption.
- **When to Use:** When initial camera poses are unavailable (COLMAP-free) and relative pose estimation between consecutive or nearby frames is required.
- **When Not to Use:** When global consistency across a large scene is needed in a single step, or when high-precision intrinsics are unknown and cannot be estimated separately.

## Background
Local 3DGS Optimization was introduced in the supplementary material of "COLMAP-Free 3D Gaussian Splatting" by Zhou et al. (CVPR 2024) [S1]. It addresses the challenge of view synthesis and pose estimation without relying on Structure-from-Motion (SfM) pipelines like COLMAP. The method builds on the principles of 3D Gaussian Splatting [S1] and adapts them for local, iterative pose refinement. It was evaluated on datasets such as CO3D-V2 and Tanks and Temples, demonstrating robustness to large camera rotations [S1].

## Sources
[S1] COLMAP-Free 3D Gaussian Splatting, https://doi.org/10.1109/cvpr52733.2024.01965, fetched 2026-10-26
