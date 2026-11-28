# SLAM and Tracking

Also called: Gaussian SLAM, 3DGS SLAM, Differentiable Gaussian SLAM

## Overview
SLAM and Tracking in the context of 3D Gaussian Splatting refers to Simultaneous Localization and Mapping methods that estimate camera poses and reconstruct dynamic or static scenes using a set of 3D Gaussians as the scene representation . Unlike standard offline 3DGS optimization which assumes known poses, these methods jointly optimize camera trajectories and Gaussian attributes (position, covariance, opacity, color) in an online or semi-online manner [S1]. What sets this family apart from sibling methods like [3DGS Initialization Policy](1398-3d-gaussian-initialization-policy.md) or [Density Control and Densification](5337-density-control-and-densification.md) is the explicit coupling of pose estimation (tracking) with scene reconstruction (mapping), often leveraging differentiable rendering to backpropagate photometric and geometric errors to the camera pose .

## Shared principles
All methods in this family share the core mechanism of representing the scene as a collection of $N$ 3D Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$, where each Gaussian $g_i$ is defined by a mean position $\boldsymbol{\mu}_i \in \mathbb{R}^3$, a covariance matrix $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$ (parameterized by rotation $R_i \in SO(3)$ and scale $S_i \in \mathbb{R}^3$), an opacity $\alpha_i \in [0, 1]$, and spherical harmonics coefficients $c_{i,lm}$ for view-dependent color [S1][G].

The rendering process uses differentiable volume rendering (alpha blending) to project these Gaussians onto the 2D image plane, producing a rendered color image $I_{rendered}$ and often a depth or opacity buffer [S1]. The optimization loop minimizes a loss function $L$ that typically combines photometric and geometric terms:
$$ L = L_{rgb} + \lambda L_{depth} + L_{reg} $$
where $L_{rgb}$ measures the difference between rendered and observed RGB images (e.g., $L_1$ or LPIPS loss), $L_{depth}$ measures the difference between rendered and observed depth (if available), and $L_{reg}$ includes regularization terms such as isotropic scale penalties [S1].

Tracking is performed by optimizing the camera pose $T_{CW}$ to minimize the reprojection error, while mapping involves updating the Gaussian parameters $\mathcal{G}$ . Keyframe management is a common sub-component to limit computational cost by maintaining a sliding window of keyframes $W_k$ for joint optimization, discarding redundant frames based on covisibility or geometric displacement [S1].

## Subtopics

### [WildGS-SLAM](5705-wildgs-slam.md)
WildGS-SLAM is a robust monocular RGB 3D Gaussian Splatting SLAM system designed for dynamic "in-the-wild" scenes. It solves the problem of tracking and mapping failure caused by dynamic distractors (moving objects/people) by making the optimization residual uncertainty-aware. Unlike standard Gaussian SLAM methods that treat all pixels as static map constraints, WildGS-SLAM uses a learned uncertainty map to weight photometric and geometric errors, effectively suppressing unreliable observations from moving objects during bundle adjustment. See [WildGS-SLAM](5705-wildgs-slam.md).

### [GS-SLAM](9192-gs-slam.md)
GS-SLAM is a Simultaneous Localization and Mapping (SLAM) method that simultaneously estimates camera poses and reconstructs 3D scenes using 3D Gaussians. It distinguishes itself from other radiance-field SLAM approaches by employing an adaptive 3D Gaussian expanding strategy to dynamically add new Gaussians and delete unreliable ones based on captured depth and rendered opacity. See [GS-SLAM](9192-gs-slam.md).

### [SplaTAM](9193-splatam.md)
SplaTAM is an RGB-D Simultaneous Localization and Mapping (SLAM) method that represents the world as a collection of 3D Gaussians. It solves the problem of online tracking and mapping by using depth and photometric Iterative Closest Point (ICP) for pose estimation and densification. It is distinguished from sibling methods like Gaussian-SLAM and Gaussian Splatting SLAM by its specific use of view-independent colors and a densification mask to prevent duplicate Gaussian creation. See [SplaTAM](9193-splatam.md).

### [Gaussian-SLAM](9195-gaussian-slam.md)
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that represents scenes as 3D Gaussians. It solves real-time photorealistic mapping and localization by fusing sub-maps derived from keyframes, addressing drift and consistency issues in online streaming approaches. It distinguishes itself from siblings like SplaTAM (which uses online densification and depth/photometric ICP) and MonoGS (which is monocular) by employing a sub-map fusion strategy to maintain global consistency. See [Gaussian-SLAM](9195-gaussian-slam.md).

### [Gaussian Splatting SLAM](10078-gaussian-splatting-slam.md)
Gaussian Splatting SLAM is a simultaneous localization and mapping (SLAM) method that reconstructs a scene as a collection of 3D Gaussians while estimating camera poses, building on [3D Gaussian Splatting Optimization](15143-3d-gaussian-splatting-optimization.md). It differs from siblings like [SplaTAM](9193-splatam.md) and [Gaussian-SLAM](9195-gaussian-slam.md) by explicitly minimizing a combined photometric and geometric residual for both tracking and mapping, and by applying isotropic scale regularization to stabilize localization. See [Gaussian Splatting SLAM](10078-gaussian-splatting-slam.md).

### [Keyframe Management](10081-keyframe-management.md)
Keyframe Management maintains a sparse sliding window $W_k$ of keyframes to enable efficient joint optimization of camera poses and 3D Gaussian attributes in online SLAM systems. It solves the problem of computational infeasibility when processing full video streams by discarding redundant frames with low overlap while ensuring sufficient parallax for robust pose estimation. Unlike fixed-interval strategies, this method uses explicit covisibility metrics and geometric displacement to dynamically insert and remove keyframes. See [Keyframe Management](10081-keyframe-management.md).

### GPS-SLAM
GPS-SLAM is an RGB-D SLAM method that represents a scene with a truncated SDF volume and a thin layer of 3D Gaussians near the surface for high-frequency appearance residuals [S1]. At render time, it raycasts the SDF for depth plus a base color, then splats only the surface-near Gaussians with SDF-guided depth culling and order-independent blending, and mixes the two images, which avoids expensive NeRF-style volumetric sampling and also removes the Gaussian depth sorting step [S1].

## Background
The integration of 3D Gaussian Splatting with SLAM emerged as a natural extension of the original 3DGS paper . While standard 3DGS requires known camera poses for offline optimization, real-world applications necessitate online reconstruction. Early works like SplaTAM and Gaussian-SLAM adapted the differentiable rasterizer to enable pose estimation via gradient descent or ICP . The field has since diversified into monocular (WildGS-SLAM), RGB-D (GS-SLAM, GPS-SLAM), and hybrid representations (GPS-SLAM) to address challenges such as dynamic scenes, scale ambiguity, and computational efficiency [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
