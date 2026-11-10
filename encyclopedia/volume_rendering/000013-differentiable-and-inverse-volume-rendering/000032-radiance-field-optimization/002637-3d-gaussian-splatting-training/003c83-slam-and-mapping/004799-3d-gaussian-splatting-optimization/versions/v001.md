# 3D Gaussian Splatting optimization

Also called: 3DGS optimization, Gaussian SLAM optimization

## Overview
This family of methods optimizes a scene representation composed of 3D Gaussians and camera poses to minimize the difference between rendered outputs and captured sensor data, enabling simultaneous localization and mapping (SLAM) [Parent](9268-slam-and-mapping.md). Unlike batch-only optimization, these methods integrate tracking and mapping in a streaming or keyframe-based loop, distinguishing themselves by specific strategies to handle densification, map consistency, and Gaussian degeneracy.

## Shared principles
All methods in this family share the core objective of optimizing a set of 3D Gaussians $\mathcal{G} = \{\mu_i, \Sigma_i, \alpha_i, c_i\}$ and camera poses $\mathbf{T}_t$ to minimize the difference between rendered outputs and captured sensor data.

**Scene Representation:**
The scene is represented as a collection of 3D Gaussians. Each Gaussian $i$ is defined by:
*   Mean position $\mu_i \in \mathbb{R}^3$.
*   Covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$, parameterized by scale $\mathbf{s}_i \in \mathbb{R}^3$ and rotation $\mathbf{R}_i \in SO(3)$.
*   Opacity $\alpha_i \in [0, 1]$.
*   Color $c_i$, which may be view-dependent (Spherical Harmonics) or view-independent.

**Optimization Objective:**
The total loss $L$ typically combines photometric and geometric terms:
$$ L = L_{photo} + \lambda L_{depth} + L_{reg} $$
where:
*   $L_{photo} = || I_{rendered} - I_{captured} ||_1$ measures color difference.
*   $L_{depth} = || D_{rendered} - D_{captured} ||_1$ measures depth difference (if depth is available).
*   $L_{reg}$ includes regularization terms such as scale isotropy to prevent degenerate Gaussians.

**Tracking and Mapping Loop:**
1.  **Initialization:** Gaussians are initialized from the first frame's depth or point cloud.
2.  **Tracking:** Camera pose $\mathbf{T}_t$ is estimated relative to the current map, often using Iterative Closest Point (ICP) or photometric optimization.
3.  **Rendering:** The current view is rendered using differentiable rasterization.
4.  **Densification/Mapping:** New Gaussians are added or existing ones are pruned based on reconstruction error, depth consistency, or uncertainty.
5.  **Optimization:** Parameters are updated via gradient descent.

**Key Challenges:**
*   **Duplicate Densification:** Avoiding the creation of redundant Gaussians in already well-reconstructed areas.
*   **Map Drift:** Accumulation of pose errors over time, mitigated by loop closure or global optimization.
*   **Degeneracy:** Gaussians becoming overly anisotropic, requiring scale regularization.

## Subtopics

### [SplaTAM](9193-splatam.md)
SplaTAM is an RGB-D Simultaneous Localization and Mapping (SLAM) method that represents the world as a collection of 3D Gaussians, enabling online tracking and mapping simultaneously. It distinguishes itself from sibling methods like Gaussian-SLAM by avoiding duplicate densification through the use of view-independent colors and a specific densification mask based on depth consistency.

### [Gaussian-SLAM](9195-gaussian-slam.md)
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that utilizes 3D Gaussian Splatting for scene representation and performs sub-map fusion to maintain global consistency. It distinguishes itself from online streaming variants like [SplaTAM](9193-splatam.md) by employing a batch-optimization approach with explicit map management, and it stabilizes the optimization process through specific scale regularization, setting it apart from standard [3D Gaussian Splatting Training](10077-3d-gaussian-splatting-training.md) pipelines.

### [Gaussian Splatting SLAM](10078-gaussian-splatting-slam.md)
Gaussian Splatting SLAM is a SLAM method that stabilizes localization and mapping by applying an extra scale regularization loss on the scale of Gaussians to encourage isotropic Gaussians.

### GS-SLAM
GS-SLAM is a SLAM method that proposes an adaptive 3D Gaussian expanding strategy to add new 3D Gaussians into the training stage and delete unreliable ones with captured depths and rendered opacity values [S1]. On the Replica dataset, GS-SLAM achieves PSNR 31.56, SSIM 0.97, and LPIPS 0.094 [S1].

## Background
3D Gaussian Splatting SLAM methods emerged as a response to the limitations of implicit representation-based methods, which assume a continuous field where surfaces are easily determined [S1]. Early works like GS-SLAM [180] introduced adaptive expanding strategies, while SplaTAM [181] addressed duplicate densification using view-independent colors and depth-based masks [S1]. To stabilize localization, methods like GaussianSplattingSLAM [182] and Gaussian-SLAM [183] introduced scale regularization to encourage isotropic Gaussians [S1]. Further developments include LIVGaussMap [184], which initializes Gaussians with LiDAR point clouds, and methods like SGS-SLAM [185], NEDS-SLAM [186], and SemGauss-SLAM [187] that extend the framework [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
