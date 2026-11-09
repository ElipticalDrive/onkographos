# SplaTAM

Also called: SplaTAM: RGB-D SLAM and Mapping with 3D Gaussians

## Overview
SplaTAM is an RGB-D Simultaneous Localization and Mapping (SLAM) method that represents the world as a collection of 3D Gaussians, enabling online tracking and mapping simultaneously [S2, unofficial]. It distinguishes itself from sibling methods like Gaussian-SLAM by avoiding duplicate densification through the use of view-independent colors and a specific densification mask based on depth consistency [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** RGB-D image frames $(I_t, D_t)$, initial camera pose, and an initial set of 3D Gaussians.
*   **Outputs:** Camera trajectory, updated 3D Gaussian map (positions, covariances, opacities, view-independent colors).

**Data Structures:**
*   **3D Gaussian:** Defined by position $\mu$, covariance matrix $\Sigma$ (or scale/rotation), opacity $\alpha$, and view-independent color $c$.
*   **Densification Mask:** A binary mask per frame indicating pixels requiring new Gaussians.

**Algorithm Steps:**
1.  **Initialization:** Initialize Gaussians from the first frame's depth data [S2, unofficial].
2.  **Tracking:** Estimate current camera pose using depth and photometric Iterative Closest Point (ICP) against the existing Gaussian map [S2, unofficial].
3.  **Rendering:** Render the current frame from the estimated pose using differentiable rasterization of the 3D Gaussians to obtain rendered RGB and Depth [S2, unofficial].
4.  **Densification Decision:** Create a densification mask by comparing the captured depth $D_{captured}$ with the rendered depth $D_{rendered}$ and the positions of current Gaussians [S1]. Pixels with significant discrepancy trigger densification.
5.  **Densification:** Add new Gaussians in regions identified by the mask [S1].
6.  **Optimization:** Optimize Gaussian parameters (position, scale, rotation, opacity, color) and camera pose to minimize photometric and depth reconstruction losses [S2, unofficial].

**Formulas:**
*   **Photometric Loss:** $L_{photo} = || I_{rendered} - I_{captured} ||_1$
*   **Depth Loss:** $L_{depth} = || D_{rendered} - D_{captured} ||_1$
*   **Total Loss:** $L = L_{photo} + \lambda L_{depth}$

**Parameters:**
*   $\lambda$: Weight for depth loss. Not given in the sources.

**Edge Cases:**
*   **Duplicate Densification:** Avoided by using view-independent colors and the specific mask logic [S1].
*   **Precision:** Relies on accurate depth sensors for initialization and tracking [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def splatam_loop(frames):
    gaussians = initialize_from_depth(frames[0].depth)
    pose = identity_matrix()
    
    for frame in frames:
        # 1. Tracking via Depth and Photometric ICP
        pose = icp_track(gaussians, frame.rgb, frame.depth, pose)
        
        # 2. Render current view
        rendered_rgb, rendered_depth = render(gaussians, pose)
        
        # 3. Determine densification mask
        # Mask is true where captured depth differs significantly from rendered depth
        # and considers current Gaussian positions [S1]
        densify_mask = compute_densification_mask(
            frame.depth, 
            rendered_depth, 
            gaussians.positions
        )
        
        # 4. Densify if needed
        if has_true_pixels(densify_mask):
            gaussians = densify(gaussians, frame, densify_mask)
            
        # 5. Optimize Gaussians and Pose
        loss = compute_loss(rendered_rgb, frame.rgb, rendered_depth, frame.depth)
        optimize(gaussians, pose, loss)
        
    return gaussians, pose
```

## Subtopics

### Gaussian-SLAM
Keyframe-based with sub-map fusion. Uses scale regularization for isotropic Gaussians. [Link](9195-gaussian-slam.md)

### Mapping Strategies
General strategies for Gaussian map management. [Link](9321-mapping-strategies.md)

### Gaussian Splatting SLAM
Monocular RGB-only with online optimization. [Link](10078-gaussian-splatting-slam.md)

## Operating characteristics
*   **Performance:** On the Replica dataset, SplaTAM achieves PSNR 34.11, SSIM 0.97, and LPIPS 0.10 [S1].
*   **Strengths:** Online mapping, real-time capability, robust tracking via ICP.
*   **Limitations:** Requires RGB-D input. Performance depends on depth accuracy.
*   **Scaling:** Scales with number of Gaussians; densification increases memory and compute.

## Background
SplaTAM was introduced as an RGB-D SLAM method that maps the world as 3D Gaussians, using depth and photometric ICP to track and densify online [S2, unofficial]. It is cited in recent advances surveys as a key example of 3DGS being used for real geometry representation, allowing photorealistic rendering, ICP-style tracking, and local editing [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
