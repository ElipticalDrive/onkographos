# SplaTAM

Also called: SplaTAM System

## Overview
SplaTAM is an RGB-D Simultaneous Localization and Mapping (SLAM) method that represents the world as a collection of 3D Gaussians [S2, unofficial]. It solves the problem of online tracking and mapping by using depth and photometric Iterative Closest Point (ICP) for pose estimation and densification [S3, unofficial]. It is distinguished from sibling methods like Gaussian-SLAM and Gaussian Splatting SLAM by its specific use of view-independent colors and a densification mask to prevent duplicate Gaussian creation [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** RGB-D image frames [S2, unofficial].
*   **Outputs:** Camera trajectory and 3D Gaussian map [S2, unofficial].

**Data Structures:**
*   **Representation:** 3D Gaussians [S2, unofficial].
*   **Densification Mask:** Binary mask indicating regions in the new frame that require new Gaussians based on the discrepancy between current Gaussians and captured depth [S1].
*   Gaussian parameters (position, scale, opacity, color): Not given in the sources.

**Steps:**
1.  **Initialization:** Not given in the sources.
2.  **Tracking:** Estimate the current camera pose using depth and photometric ICP against the existing Gaussian map [S2, unofficial].
3.  **Rendering:** Render the current frame from the estimated pose using differentiable Gaussian rasterization to obtain rendered depth and color [G].
4.  **Densification Decision:** Compute a densification mask by comparing the rendered depth/opacity with the captured depth of the new frame [S1].
5.  **Densification:** Add new Gaussians in regions indicated by the mask where current Gaussians do not adequately cover the captured depth [S1].
6.  **Optimization:** Optimize Gaussian parameters and camera pose using photometric and depth losses [S2, unofficial].
7.  **Pruning:** Not given in the sources.

**Formulas:**
*   Photometric and depth loss formulations: Not given in the sources.

**Parameters:**
*   Depth loss weight ($\lambda$): Not given in the sources.
*   Densification threshold: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Duplicate Densification:** SplaTAM avoids this by using view-independent colors and a specific densification mask [S1].
*   **Precision:** Standard floating-point precision is assumed. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splatam_step(rgb_frame, depth_frame, gaussian_map, camera_pose):
    # 1. Tracking: Estimate pose using depth and photometric ICP
    new_camera_pose = minimize_icp_residuals(gaussian_map, rgb_frame, depth_frame)
    
    # 2. Rendering: Alpha-blend projected Gaussians to obtain rendered frame
    rendered_depth, rendered_color = alpha_blend_gaussians(gaussian_map, new_camera_pose)
    
    # 3. Densification Decision: Compute mask based on depth/opacity discrepancy
    densification_mask = compute_depth_discrepancy_mask(rendered_depth, depth_frame, gaussian_map)
    
    # 4. Densification: Spawn new Gaussians at masked pixel locations
    gaussian_map = spawn_gaussians(gaussian_map, rgb_frame, depth_frame, densification_mask)
    
    # 5. Optimization: Update Gaussian parameters and pose via gradient descent
    gaussian_map, new_camera_pose = gradient_descent_step(gaussian_map, new_camera_pose, rgb_frame, depth_frame)
    
    return gaussian_map, new_camera_pose

# Standard operations:
# minimize_icp_residuals: iteratively aligns rendered depth/color to captured frame
# alpha_blend_gaussians: sorts Gaussians by depth, alpha-blends to image plane
# compute_depth_discrepancy_mask: masks pixels where rendered depth differs from captured depth
# spawn_gaussians: adds new Gaussians at masked locations using captured depth and view-independent colors
# gradient_descent_step: updates Gaussian parameters and pose to minimize photometric/depth losses
```

## Subtopics

### Gaussian-SLAM
Keyframe-based with sub-map fusion [S2, unofficial].

### Gaussian Splatting SLAM (MonoGS)
Monocular RGB-only with online optimization [S2, unofficial].

### Differentiable Pose Estimation
Uses differentiable rendering for pose tracking [G].

## Operating characteristics
*   **Time Complexity:** Online, real-time capable. Not given in the sources.
*   **Memory Complexity:** Scales with the number of Gaussians. Not given in the sources.
*   **Measured Performance:** On the Replica dataset, SplaTAM achieves PSNR 34.11, SSIM 0.97, and LPIPS 0.10 [S1].
*   **Strengths:** Online mapping, real-time performance, high-quality reconstruction [S2, unofficial].
*   **Limitations:** Requires RGB-D input [S2, unofficial].

## Background
SplaTAM was introduced as an RGB-D SLAM method that maps the world as 3D Gaussians [S2, unofficial]. It uses depth and photometric ICP to track and densify online [S3, unofficial]. The method is cited in "Recent advances in 3D Gaussian splatting" [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] spla-tam/SplaTAM, https://github.com/spla-tam/SplaTAM, fetched 2026-10-27 (unofficial)
[S3] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
