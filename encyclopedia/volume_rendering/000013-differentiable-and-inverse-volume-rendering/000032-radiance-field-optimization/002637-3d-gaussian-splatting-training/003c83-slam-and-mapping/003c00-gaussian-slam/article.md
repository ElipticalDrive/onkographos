# Gaussian-SLAM

Also called: Yugay et al. 2024 SLAM

## Overview
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that utilizes 3D Gaussian Splatting for scene representation and performs sub-map fusion to maintain global consistency [S2, unofficial]. It distinguishes itself from online streaming variants like [SplaTAM](9193-splatam.md) by employing a batch-optimization approach with explicit map management, and it stabilizes the optimization process through specific scale regularization, setting it apart from standard [3D Gaussian Splatting Training](10077-3d-gaussian-splatting-training.md) pipelines.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of RGB images (and optionally depth, though the core definition emphasizes keyframe-based fusion) and initial camera poses.
*   **Outputs:** A global 3D Gaussian map (positions, covariances, opacities, colors) and a trajectory of camera poses.

**Data Structures:**
*   **Sub-maps:** The global map is partitioned into local sub-maps associated with keyframes. Each sub-map contains a set of 3D Gaussians.
*   **Gaussian Attributes:** Standard 3DGS attributes: mean position $\mathbf{\mu}$, covariance $\mathbf{\Sigma}$ (or rotation $\mathbf{R}$ and scale $\mathbf{s}$), opacity $\alpha$, and spherical harmonics coefficients for color.

**Algorithm Steps:**
1.  **Keyframe Selection:** Select keyframes from the input stream based on motion or overlap criteria to manage computational load and ensure coverage.
2.  **Sub-map Initialization:** For each new keyframe, initialize a local sub-map of Gaussians. This may involve densification from previous frames or initialization from depth/point clouds if available [G].
3.  **Local Optimization:** Optimize the Gaussians in the current sub-map and the camera pose of the keyframe using photometric rendering loss against the captured image.
4.  **Scale Regularization:** Apply an additional loss term to the Gaussian scales to encourage isotropy. This stabilizes localization and mapping by preventing degenerate, elongated Gaussians [S1].
    *   Let $\mathbf{s}_i$ be the scale vector of the $i$-th Gaussian. The regularization loss $L_{scale}$ can be formulated as:
        $$ L_{scale} = \sum_i \left( \frac{\max(\mathbf{s}_i) - \min(\mathbf{s}_i)}{\max(\mathbf{s}_i) + \min(\mathbf{s}_i)} \right)^2 $$
        [G] (Note: Exact formula form may vary, but the intent is minimizing the ratio of scale eigenvalues).
5.  **Sub-map Fusion:** When a new keyframe overlaps significantly with existing sub-maps, perform bundle adjustment and fusion operations to merge the Gaussians from the new sub-map into the global map, resolving conflicts and optimizing the global pose graph.
6.  **Loop Closure (Implicit):** The fusion process and global pose optimization implicitly handle loop closures by re-anchoring Gaussians and correcting drift [S2, unofficial].

**Parameters:**
*   **Keyframe Interval:** Not given in the sources.
*   **Scale Regularization Weight:** Not given in the sources.
*   **Rendering Resolution:** Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Degenerate Gaussians:** Without scale regularization, Gaussians may become highly anisotropic, leading to poor reconstruction quality and tracking instability [S1].
*   **Map Drift:** In the absence of loop closure mechanisms, drift can accumulate. The sub-map fusion strategy mitigates this but requires sufficient overlap [S2, unofficial].
*   **Initialization:** Poor initial poses or sparse keyframes can lead to failed sub-map initialization [G].

## Subtopics

### MonoGS
A monocular RGB-only SLAM method that performs online optimization using Gaussian Splatting [S2, unofficial]. Unlike the keyframe-based Gaussian-SLAM, MonoGS processes frames incrementally in an online manner, making it suitable for real-time applications where batch processing is not feasible. It relies on online densification and optimization strategies to maintain map consistency without explicit sub-map fusion [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** Depends on the number of keyframes and the size of sub-maps. Fusion operations can be computationally expensive.
*   **Memory:** Scales with the number of Gaussians in the global map. Sub-map management adds overhead compared to a single monolithic map.

**Measured Performance:**
On the Replica dataset, Gaussian-SLAM achieves a PSNR of 37.50, SSIM of 0.96, and LPIPS of 0.07 [S1]. This performance is competitive with other Gaussian-based SLAM methods like [SplaTAM](9193-splatam.md) (PSNR 34.11, SSIM 0.97, LPIPS 0.10) and [Gaussian Splatting SLAM](10078-gaussian-splatting-slam.md) (PSNR 38.90, SSIM 0.99, LPIPS 0.07) [S1].

**Strengths and Limitations:**
*   **Strengths:** High reconstruction quality; stable localization due to scale regularization; effective handling of large scenes via sub-map fusion.
*   **Limitations:** Not strictly online; requires careful keyframe selection; fusion complexity.

**When to Use:**
Use when high-quality offline or semi-offline reconstruction is required, and scene size necessitates modular map management. Avoid when strict real-time online performance is critical without post-processing.

## Background
Gaussian-SLAM was introduced by Yugay et al. in 2024 [S2, unofficial]. It represents the application of 3D Gaussian Splatting to SLAM problems, leveraging the efficiency and visual fidelity of Gaussian primitives for simultaneous localization and mapping. The method builds on the principles of [3D Gaussian Splatting Training](10077-3d-gaussian-splatting-training.md) but adapts them for incremental map building and pose estimation.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
