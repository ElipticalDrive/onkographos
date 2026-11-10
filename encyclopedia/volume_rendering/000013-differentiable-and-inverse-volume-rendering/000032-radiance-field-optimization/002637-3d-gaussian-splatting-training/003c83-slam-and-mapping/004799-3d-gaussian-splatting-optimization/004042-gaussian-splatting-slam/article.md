# Gaussian Splatting SLAM

Also called: Gaussian Splatting SLAM Tracking and Mapping Optimization, GaussianSplattingSLAM, Monocular Gaussian SLAM, RGB-D SLAM with Gaussian Splatting

## Overview
Gaussian Splatting SLAM is a simultaneous localization and mapping method that optimizes a dynamic set of 3D Gaussians and camera poses to minimize photometric and geometric residuals between rendered views and captured sensor data. It distinguishes itself from siblings like [SplaTAM](9193-splatam.md) and [Gaussian-SLAM](9195-gaussian-slam.md) by explicitly introducing an isotropic scale regularization loss to stabilize localization and mapping, and by employing specific learning rate adjustments and iteration limits for tracking and mapping phases.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Current camera pose estimate, captured RGB image, captured depth map (if available), and the current set of 3D Gaussians $\mathcal{G}$.
*   **Outputs:** Updated camera pose $\mathbf{T}$, updated Gaussian parameters $\mathcal{G}$, and keyframe selection decisions.

**Data Structures:**
The scene is represented as a collection of 3D Gaussians $\mathcal{G} = \{\mu_i, \Sigma_i, \alpha_i, c_i\}$, where $\mu_i \in \mathbb{R}^3$ is the mean position, $\Sigma_i \in \mathbb{R}^{3 \times 3}$ is the covariance matrix (parameterized by scale $\mathbf{s}_i \in \mathbb{R}^3$ and rotation $\mathbf{R}_i \in SO(3)$), $\alpha_i \in [0, 1]$ is opacity, and $c_i$ is color. Keyframes are stored with their associated visible Gaussians $\mathcal{G}_i^v$.

**Parameters:**
*   Photometric loss weight $\lambda_{pho} = 0.9$ for RGB-D experiments [S2, unofficial].
*   Isotropic regularization weight $\lambda_{iso} = 10$ for both monocular and RGB-D experiments [S2, unofficial].
*   Tracking learning rate for Gaussian positions $\mu_W$ is increased by a factor of 10 in monocular settings [S2, unofficial].
*   Camera pose learning rates: 0.003 for rotation, 0.001 for translation [S2, unofficial].

**Steps:**

1.  **Tracking:** Estimate the current camera pose $\mathbf{T}$ by minimizing the weighted sum of photometric and geometric residuals:
    $$ \min_{\mathbf{T} \in SE(3)} \lambda_{pho} E_{pho} + (1 - \lambda_{pho}) E_{geo} $$
    Perform 100 tracking iterations per frame. Terminate early if the magnitude of the pose update becomes less than $10^{-4}$ [S2, unofficial].

2.  **Keyframe Selection:** Determine if the current frame should become a keyframe based on Gaussian covisibility with existing keyframes. Calculate the Intersection of Union (IOU) and Overlap Coefficient (OC) between the current frame's visible Gaussians $\mathcal{G}_{curr}^v$ and keyframe $k$'s visible Gaussians $\mathcal{G}_k^v$:
    $$ IOU_{cov}(curr, k) = \frac{|\mathcal{G}_{curr}^v \cap \mathcal{G}_k^v|}{|\mathcal{G}_{curr}^v \cup \mathcal{G}_k^v|} $$
    $$ OC_{cov}(curr, k) = \frac{|\mathcal{G}_{curr}^v \cap \mathcal{G}_k^v|}{\min(|\mathcal{G}_{curr}^v|, |\mathcal{G}_k^v|)} $$
    If covisibility is low, add the current frame as a new keyframe [S2, unofficial].

3.  **Mapping:** Optimize the parameters of the Gaussians and the poses of the keyframes in $\mathcal{W}$ by minimizing:
    $$ \min_{\mathcal{G}, \forall k \in \mathcal{W}} \sum_{k \in \mathcal{W}} (\lambda_{pho} E_{pho}^k + (1 - \lambda_{pho}) E_{geo}^k) + \lambda_{iso} E_{iso} $$
    Perform 150 iterations for the single-process implementation [S2, unofficial]. The term $E_{iso}$ is an extra scale regularization loss that encourages isotropic Gaussians to stabilize localization and mapping [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_splatting_slam_step(current_frame, keyframes, gaussians, is_monocular):
    # 1. Tracking
    pose = current_frame.estimated_pose
    for i in range(100):
        # Render current view using differentiable rasterization
        rendered_rgb, rendered_depth = render(gaussians, pose)
        
        # Calculate photometric and geometric residuals
        e_pho = photometric_loss(rendered_rgb, current_frame.rgb)
        e_geo = geometric_loss(rendered_depth, current_frame.depth)
        
        # Minimize weighted sum
        loss = 0.9 * e_pho + 0.1 * e_geo
        pose = optimize_pose(pose, loss, lr_rot=0.003, lr_trans=0.001)
        
        # Early termination
        if pose_update_magnitude(pose) < 1e-4:
            break
            
    current_frame.pose = pose
    
    # 2. Keyframe Selection
    visible_gaussians = get_visible_gaussians(gaussians, current_frame)
    is_keyframe = False
    for kf in keyframes:
        kf_visible = get_visible_gaussians(gaussians, kf)
        iou = len(visible_gaussians & kf_visible) / len(visible_gaussians | kf_visible)
        oc = len(visible_gaussians & kf_visible) / min(len(visible_gaussians), len(kf_visible))
        if iou < threshold_iou and oc < threshold_oc:
            is_keyframe = True
            break
            
    if is_keyframe:
        keyframes.append(current_frame)
        
    # 3. Mapping
    for i in range(150):
        total_loss = 0
        for kf in keyframes:
            rendered_rgb, rendered_depth = render(gaussians, kf.pose)
            e_pho = photometric_loss(rendered_rgb, kf.rgb)
            e_geo = geometric_loss(rendered_depth, kf.depth)
            total_loss += 0.9 * e_pho + 0.1 * e_geo
            
        # Isotropic regularization
        e_iso = isotropic_regularization(gaussians)
        total_loss += 10 * e_iso
        
        # Optimize Gaussians and Keyframe Poses
        if is_monocular:
            optimize_gaussians(gaussians, total_loss, lr_pos=base_lr * 10)
        else:
            optimize_gaussians(gaussians, total_loss, lr_pos=base_lr)
        optimize_keyframes(keyframes, total_loss)
        
    return keyframes, gaussians
```

## Operating characteristics
*   **Time Complexity:** Tracking is limited to 100 iterations per frame, and mapping to 150 iterations per update. The complexity depends on the number of Gaussians and keyframes involved in the covisibility check and optimization.
*   **Memory Complexity:** Scales with the number of Gaussians and keyframes. Storing visible Gaussian sets for covisibility checks adds overhead.
*   **Performance:** On the Replica dataset, GaussianSplattingSLAM achieves a PSNR of 38.90, SSIM of 0.99, and LPIPS of 0.07 [S1].
*   **Strengths:** The isotropic regularization helps prevent degenerate Gaussians, improving stability. The specific learning rate adjustments for monocular settings aid convergence.
*   **Limitations:** The method relies on accurate initial depth or point cloud for initialization. Performance may degrade in textureless or highly reflective environments where photometric and geometric residuals are ambiguous.

## Background
Gaussian Splatting SLAM was introduced by [S2, unofficial] as a method to combine the efficiency of 3D Gaussian Splatting with the requirements of simultaneous localization and mapping. It builds on the principles of [3D Gaussian Splatting optimization](12052-3d-gaussian-splatting-optimization.md). The method was developed to address challenges in dynamic scene reconstruction and real-time SLAM, leveraging the differentiable nature of Gaussian Splatting for end-to-end optimization.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
