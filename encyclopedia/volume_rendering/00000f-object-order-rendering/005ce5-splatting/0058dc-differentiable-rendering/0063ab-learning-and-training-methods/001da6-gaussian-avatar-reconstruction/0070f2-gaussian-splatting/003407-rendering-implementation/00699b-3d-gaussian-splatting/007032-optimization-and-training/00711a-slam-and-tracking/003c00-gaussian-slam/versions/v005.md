# Gaussian-SLAM

Also called: Yugay et al. 2024 SLAM

## Overview
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that utilizes 3D Gaussian Splatting for dense scene representation [S2, unofficial]. It solves the problem of real-time, photorealistic mapping and localization by fusing sub-maps derived from keyframes, addressing the drift and consistency issues common in online streaming approaches [S2, unofficial]. It distinguishes itself from sibling methods like SplaTAM (which uses online densification and depth/photometric ICP) and MonoGS (which is monocular) by employing a sub-map fusion strategy to maintain global consistency [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** RGB image sequences, camera intrinsics, and initial pose estimates.
*   **Outputs:** A dense 3D Gaussian map, camera poses, and rendered photorealistic images.

**Data Structures:**
The method maintains a global map composed of 3D Gaussians. Each Gaussian is defined by:
*   Mean position $\mu \in \mathbb{R}^3$
*   Covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ (parameterized by rotation $R \in SO(3)$ and scale $S \in \mathbb{R}^3$)
*   Opacity $\alpha \in [0, 1]$
*   Spherical harmonics coefficients for color $c$

**Steps:**
1.  **Keyframe Selection:** Identify keyframes from the input stream based on motion or coverage criteria to form sub-maps [S2, unofficial].
2.  **Sub-map Optimization:** Optimize the Gaussians within each sub-map independently to minimize rendering loss against the corresponding keyframe images [S2, unofficial].
3.  **Sub-map Fusion:** Align and fuse the optimized sub-maps into a global map. This involves estimating the relative poses between sub-maps and merging overlapping Gaussian representations [S2, unofficial].
4.  **Scale Regularization:** Apply an extra scale regularization loss on the scale of Gaussians to encourage isotropic shapes, stabilizing localization and mapping [S1].
5.  **Pose Estimation:** Update camera poses by minimizing the photometric reprojection error between the rendered image and the observed image [S2, unofficial].

**Formulas:**
The total loss function $L$ includes the rendering loss $L_{render}$ and the scale regularization loss $L_{scale}$:
$$ L = L_{render} + \lambda L_{scale} $$
Where $L_{scale}$ penalizes deviations from isotropic scales, encouraging $\sigma_x \approx \sigma_y \approx \sigma_z$ [S1].

**Parameters:**
*   $\lambda$: Weight for scale regularization. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Degenerate Input:** Poor lighting or textureless regions may lead to ambiguous pose estimates.
*   **Precision:** Floating-point precision in covariance matrix decomposition can affect stability during optimization [G].

## Subtopics

### MonoGS
A monocular RGB-only SLAM method with online optimization using Gaussian Splatting [S2, unofficial]. It differs from Gaussian-SLAM by not relying on keyframe-based sub-map fusion, instead optimizing the map continuously online. It is chosen when depth sensors are unavailable and real-time online mapping is preferred over batch-like sub-map fusion.

### Vox-Fusion
A hybrid SLAM system combining a voxel-based neural implicit surface representation with an octree structure for dynamic scene expansion [S3]. It encodes and optimizes each scene voxel implicitly, supporting incremental mapping without prior environment knowledge and leveraging a multi-process framework for heightened speed [S3]. It demonstrates improved accuracy and completeness compared to prior methods [S3].

### Neural SLAM Pipeline
An end-to-end simultaneous localization and mapping approach that produces dense, photorealistic maps while tracking camera pose, replacing sparse classical SLAM outputs [S4, unofficial]. The pipeline involves: 1. Tracking the current frame against the existing map. 2. Updating the map with new views by inserting new Gaussians where coverage is poor and refining existing ones. 3. Optional loop closure to detect revisited locations and correct drift. 4. Continuously rendering the photorealistic map [S4, unofficial]. It is dense and photorealistic, supporting relighting and rendering downstream, but requires heavier compute and is less mature for real-time on edge hardware compared to classical SLAM [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** On the Replica dataset, Gaussian-SLAM achieves a PSNR of 37.50, SSIM of 0.96, and LPIPS of 0.07 [S1].
*   **Strengths:** Provides dense, photorealistic maps and supports downstream rendering tasks [S4, unofficial]. The sub-map fusion approach helps maintain global consistency [S2, unofficial].
*   **Limitations:** Heavier compute requirements compared to classical SLAM [S4, unofficial]. Less mature for real-time deployment on edge hardware [S4, unofficial].
*   **When to Use:** When photorealistic dense mapping is required and computational resources are sufficient.
*   **When Not to Use:** When real-time performance on resource-constrained devices is critical.

## Background
Gaussian-SLAM was introduced by Yugay et al. in 2024 [S2, unofficial]. It builds on the principles of 3D Gaussian Splatting, leveraging its differentiable rendering capabilities for SLAM tasks. The method represents a shift from sparse classical SLAM outputs to dense, neural-renderable maps, addressing the limitations of earlier NeRF-based SLAM systems in terms of editing and real-time performance [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
