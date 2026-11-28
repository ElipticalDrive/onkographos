# Gaussian Splatting SLAM

Also called: Gaussian Splatting SLAM Tracking and Mapping Optimization, GaussianSplattingSLAM, Monocular Gaussian SLAM, RGB-D SLAM with Gaussian Splatting

## Overview
Gaussian Splatting SLAM is a simultaneous localization and mapping method that represents the scene as a dynamic collection of 3D Gaussians, optimizing both camera poses and Gaussian parameters jointly to reconstruct high-fidelity environments [S2, unofficial]. It distinguishes itself from sibling methods like [SplaTAM](9193-splatam.md) and [GS-SLAM](9192-gs-slam.md) by explicitly incorporating an isotropic scale regularization term to stabilize the optimization of Gaussian shapes, which is critical for robust tracking in challenging scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current camera pose estimate $T_{CW}$, current Gaussian map $\mathcal{G}$, and new sensor data (RGB image $I_{rgb}$ and optionally depth map $D_{depth}$).
*   **Outputs:** Updated camera pose $T_{CW}'$, updated Gaussian map $\mathcal{G}'$, and keyframe list management.

**Data Structures:**
The Gaussian map $\mathcal{G}$ consists of $N$ Gaussians. Each Gaussian $g_i$ is defined by:
*   Mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$.
*   Covariance matrix $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$, parameterized by rotation $R_i \in SO(3)$ and scale $S_i \in \mathbb{R}^3$ such that $\boldsymbol{\Sigma}_i = R_i S_i S_i^T R_i^T$.
*   Opacity $\alpha_i \in [0, 1]$.
*   Spherical harmonics coefficients $c_{i,lm}$ for view-dependent color.
*   Visible Gaussians for a keyframe $k$, denoted $\mathcal{G}_k^v$, are stored to facilitate covisibility checks.

**Algorithm Steps:**
The method alternates between Tracking and Mapping phases.

1.  **Tracking:**
    Optimize the camera pose $T_{CW}$ for the current frame relative to the existing keyframe map. The objective minimizes a weighted sum of photometric and geometric residuals [S2, unofficial]:
    $$ \min_{T \in SE(3)} \lambda_{pho} E_{pho} + (1 - \lambda_{pho}) E_{geo} $$
    where $E_{pho}$ is the photometric error (difference between rendered and observed RGB) and $E_{geo}$ is the geometric error (difference between rendered depth and observed depth, if available) [S2, unofficial].
    *   Perform up to 100 iterations of optimization [S2, unofficial].
    *   Terminate early if the magnitude of the pose update $\|\Delta T\| < 10^{-4}$ [S2, unofficial].
    *   In monocular settings, increase the learning rate of Gaussian positions $\boldsymbol{\mu}_W$ by a factor of 10 to aid initialization [S2, unofficial].

2.  **Mapping:**
    Optimize the Gaussian parameters $\mathcal{G}$ and potentially the poses of recent keyframes $T_{CW}^k$ for a sliding window $\mathcal{W}$ [S2, unofficial]. The objective is [S2, unofficial]:
    $$ \min_{\mathcal{G}, T_{CW}^k \forall k \in \mathcal{W}} \sum_{k \in \mathcal{W}} (\lambda_{pho} E_{pho}^k + (1 - \lambda_{pho}) E_{geo}^k) + \lambda_{iso} E_{iso} $$
    where $E_{iso}$ is the isotropic regularization loss [S1].
    *   Perform 150 iterations for single-process implementations [S2, unofficial].
    *   Apply isotropic regularization $E_{iso}$ to encourage spherical Gaussians, stabilizing localization and mapping [S1].

3.  **Keyframe Management:**
    Select new keyframes based on Gaussian covisibility to limit computational cost [S2, unofficial]. For two keyframes $i$ and $j$, compute:
    $$ IOU_{cov}(i, j) = \frac{|\mathcal{G}_i^v \cap \mathcal{G}_j^v|}{|\mathcal{G}_i^v \cup \mathcal{G}_j^v|} $$
    $$ OC_{cov}(i, j) = \frac{|\mathcal{G}_i \cap \mathcal{G}_j|}{\min(|\mathcal{G}_i^v|, |\mathcal{G}_j^v|)} $$
    where $\mathcal{G}_i^v$ is the set of Gaussians visible in keyframe $i$ [S2, unofficial]. A new frame is added as a keyframe if the covisibility with existing keyframes falls below a threshold.

**Parameters:**
*   Photometric loss weight $\lambda_{pho} = 0.9$ for RGB-D experiments [S2, unofficial].
*   Isotropic regularization weight $\lambda_{iso} = 10$ for both monocular and RGB-D experiments [S2, unofficial].
*   Camera pose learning rates: 0.003 for rotation, 0.001 for translation [S2, unofficial].
*   Gaussian learning rates: Default from original Gaussian Splatting [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_splatting_slam(current_rgb, current_depth, current_pose, gaussian_map, keyframes):
    # 1. Tracking Phase
    # Optimize current camera pose T_CW
    pose = current_pose
    for i in range(100):
        # Render current view using gaussian_map and pose
        rendered_rgb, rendered_depth = render(gaussian_map, pose)
        
        # Calculate residuals
        e_pho = photometric_loss(rendered_rgb, current_rgb)
        e_geo = geometric_loss(rendered_depth, current_depth) if current_depth is not None else 0
        
        # Combined loss for tracking
        loss = 0.9 * e_pho + 0.1 * e_geo
        
        # Gradient descent step for pose
        delta_pose = optimize_gradient(loss, pose)
        pose = compose(pose, delta_pose)
        
        # Early termination
        if magnitude(delta_pose) < 1e-4:
            break
            
    current_pose = pose
    
    # 2. Mapping Phase
    # Optimize Gaussians and keyframe poses in window W
    # Note: In single-process, 150 iterations are used [S2, unofficial]
    for i in range(150):
        total_loss = 0
        for kf in keyframes:
            rendered_rgb, rendered_depth = render(gaussian_map, kf.pose)
            e_pho = photometric_loss(rendered_rgb, kf.rgb)
            e_geo = geometric_loss(rendered_depth, kf.depth) if kf.depth is not None else 0
            
            # Mapping loss includes isotropic regularization [S1]
            e_iso = isotropic_regularization(gaussian_map)
            total_loss += 0.9 * e_pho + 0.1 * e_geo + 10 * e_iso
            
        # Gradient descent step for Gaussians and keyframe poses
        update_gaussians_and_poses(total_loss)
        
    # 3. Keyframe Selection
    # Check covisibility with existing keyframes [S2, unofficial]
    if should_add_keyframe(gaussian_map, keyframes):
        add_keyframe(current_rgb, current_depth, current_pose, gaussian_map, keyframes)
        
    return current_pose, gaussian_map
```

## Operating characteristics
*   **Complexity:** Tracking is $O(N)$ per iteration where $N$ is the number of visible Gaussians. Mapping is $O(N \cdot K)$ where $K$ is the number of keyframes in the window.
*   **Performance:** On the Replica dataset, GaussianSplattingSLAM achieves a PSNR of 38.90, SSIM of 0.99, and LPIPS of 0.07 [S1].
*   **Strengths:** High-fidelity reconstruction due to explicit Gaussian representation; robust tracking via isotropic regularization [S1].
*   **Limitations:** Computational cost increases with the number of Gaussians and keyframes; requires careful tuning of learning rates, especially in monocular settings [S2, unofficial].

## Background
Gaussian Splatting SLAM builds on the principles of 3D Gaussian Splatting [S1]. The specific method detailed here, often referred to as GaussianSplattingSLAM, was introduced in a preprint by [S2, unofficial]. It addresses the need for real-time, high-quality SLAM by leveraging the differentiable nature of Gaussian splatting.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
