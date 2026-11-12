# Gaussian-SLAM

Also called: Yugay et al. 2024 SLAM

## Overview
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that represents the scene as a collection of 3D Gaussians and employs sub-map fusion to maintain consistency [S2, unofficial]. It distinguishes itself from sibling methods like SplaTAM, which uses continuous depth and photometric ICP for online tracking, by structuring the map into discrete keyframes that are fused, offering a robust alternative for scenarios requiring explicit keyframe management [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** RGB-D image sequences (or RGB with estimated depth), initial camera poses.
*   **Outputs:** Optimized camera poses $\mathbf{T}_t$, a set of 3D Gaussians $\mathcal{G}$ representing the scene, and a fused global map.

**Data Structures:**
The scene is represented as a set of 3D Gaussians $\mathcal{G} = \{\mu_i, \Sigma_i, \alpha_i, c_i\}$, where $\mu_i$ is the mean, $\Sigma_i$ is the covariance (parameterized by scale $\mathbf{s}_i$ and rotation $\mathbf{R}_i$), $\alpha_i$ is opacity, and $c_i$ is color [S1]. The map is organized into sub-maps associated with keyframes.

**Optimization Objective:**
The total loss $L$ minimizes the difference between rendered and captured data, with a specific emphasis on stabilizing localization:
$$ L = L_{photo} + \lambda L_{depth} + L_{reg} $$
where $L_{photo}$ is the photometric loss, $L_{depth}$ is the depth loss, and $L_{reg}$ includes an extra scale regularization term [S1].

**Steps:**
1.  **Initialization:** Initialize Gaussians from the first frame's depth or point cloud [S1].
2.  **Tracking:** Estimate the current camera pose $\mathbf{T}_t$ relative to the existing map.
3.  **Keyframe Selection:** Determine if the current view constitutes a new keyframe based on motion or reconstruction error.
4.  **Sub-map Fusion:** When a new keyframe is added, fuse its local Gaussian sub-map into the global map, resolving overlaps and redundancies [S2, unofficial].
5.  **Densification:** Add new Gaussians in areas with high reconstruction error or low opacity [S1].
6.  **Regularization:** Apply scale regularization to encourage isotropic Gaussians, preventing degenerate shapes and stabilizing the optimization [S1].
7.  **Optimization:** Update Gaussian parameters and camera poses via gradient descent [S1].

**Parameters:**
*   $\lambda$: Weight for depth loss. Not given in the sources.
*   Scale regularization weight: Not given in the sources.

**Edge Cases:**
*   **Degeneracy:** Gaussians may become overly anisotropic; mitigated by scale regularization [S1].
*   **Redundancy:** Duplicate Gaussians in well-reconstructed areas; mitigated by pruning and fusion strategies [S1].

## Subtopics

### MonoGS
A monocular RGB-only SLAM method that performs online optimization using Gaussian Splatting without depth sensors [S2, unofficial]. It relies on geometric cues and optical flow to estimate depth implicitly, making it suitable for standard camera setups where depth data is unavailable.

### Vox-Fusion
A hybrid SLAM system that combines a voxel-based neural implicit surface representation with an octree structure for dynamic scene expansion [S3]. It encodes and optimizes each scene voxel implicitly, supporting incremental mapping without prior environment knowledge and leveraging a multi-process framework for speed [S3].

## Operating characteristics
On the Replica dataset, Gaussian-SLAM achieves a PSNR of 37.50, SSIM of 0.96, and LPIPS of 0.07 [S1]. The method's reliance on keyframe-based sub-map fusion may introduce latency during the fusion step compared to purely online methods, but it offers improved global consistency. The scale regularization adds computational overhead but is critical for preventing map drift and degeneracy [S1].

## Background
Gaussian-SLAM was introduced by Yugay et al. in 2024 [S2, unofficial]. It emerged as part of the broader adoption of 3D Gaussian Splatting for SLAM, leveraging the efficiency and differentiability of Gaussian representations to overcome limitations of earlier NeRF-based SLAM systems, such as slow training times and difficulty in local editing [S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
