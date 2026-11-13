# 3D Gaussian Splatting optimization
Also called: 3DGS optimization, Gaussian SLAM optimization

## Overview
This family encompasses methods that jointly optimize camera poses and a 3D Gaussian scene representation from image streams, solving the problem of simultaneous localization and mapping (SLAM) or pose estimation in dynamic or blurry scenarios. It distinguishes itself from sibling methods like [Mapping Strategies](9321-mapping-strategies.md) and [Semantic-aware mapping](12053-semantic-aware-mapping.md) by explicitly handling the differentiable rendering pipeline's gradients for pose updates, and from [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md) by using explicit Gaussian primitives rather than implicit neural fields.

## Shared principles
The core mechanism involves a coupled optimization loop where the scene representation $\mathcal{R}$ (a set of 3D Gaussians $\mathcal{G}$) and camera poses $\mathbf{T}_t$ are jointly or alternately optimized. Each Gaussian $i$ is defined by mean position $\mu_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (parameterized by scale $\mathbf{s}_i \in \mathbb{R}^3$ and rotation $\mathbf{R}_i \in SO(3)$), opacity $\alpha_i \in [0, 1]$, and color $c_i$.

The optimization objective minimizes a loss function $L$:
$$ L = L_{photo} + \lambda L_{depth} + L_{reg} $$
where $L_{photo} = || I_{rendered} - I_{captured} ||_1$ measures color difference, $L_{depth} = || D_{rendered} - D_{captured} ||_1$ measures depth difference (if available), and $L_{reg}$ includes regularization terms such as scale isotropy to prevent degenerate Gaussians.

Key challenges include avoiding duplicate densification in well-reconstructed areas, mitigating map drift from accumulated pose errors, and preventing Gaussian degeneracy (overly anisotropic scales) through regularization.

## Subtopics

### [SplaTAM](9193-splatam.md)
SplaTAM is an RGB-D Simultaneous Localization and Mapping (SLAM) method that represents the scene as a 3D Gaussian Splatting (3DGS) model. It solves the problem of online mapping and tracking by jointly optimizing camera poses and Gaussian parameters using depth and photometric constraints. It is distinguished from sibling methods like Gaussian-SLAM and Gaussian Splatting SLAM by its use of view-independent colors and a specific densification mask strategy to prevent duplicate Gaussian creation.

### [Gaussian-SLAM](9195-gaussian-slam.md)
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that constructs a scene representation using 3D Gaussians and employs sub-map fusion to manage global consistency. It distinguishes itself from sibling methods like SplaTAM, which relies on depth-assisted ICP for tracking, by utilizing a specific scale regularization loss to enforce isotropic Gaussians, thereby stabilizing the joint optimization of camera poses and map geometry.

### [Gaussian Splatting SLAM](10078-gaussian-splatting-slam.md)
Gaussian Splatting SLAM is a simultaneous localization and mapping method that jointly optimizes camera poses and a 3D Gaussian scene representation from monocular or RGB-D video streams. It distinguishes itself from siblings like [SplaTAM](9193-splatam.md) and [Gaussian-SLAM](9195-gaussian-slam.md) by explicitly enforcing isotropic scale regularization to stabilize localization and mapping, and by utilizing a covisibility-based keyframing strategy for efficient map management.

### [Differentiable Pose Estimation](12756-differentiable-pose-estimation.md)
Differentiable pose estimation optimizes camera poses by back-propagating photometric loss gradients through a 3D Gaussian Splatting (3DGS) renderer. It solves the problem of recovering accurate camera motion, particularly in scenarios involving motion blur, by treating pose as a differentiable variable within the rendering pipeline. It sets itself apart from sibling methods like SplaTAM and Gaussian-SLAM by explicitly decoupling the pose optimization from the scene geometry update in specific implementations (e.g., Deblur-GS), utilizing a simplified gradient derivation that ignores covariance and color terms to ensure stability during short optimization steps.

### GS-SLAM
#### definition
GS-SLAM is a SLAM method that proposes an adaptive 3D Gaussian expanding strategy to add new 3D Gaussians into the training stage and delete unreliable ones with captured depths and rendered opacity values [S1]. To avoid duplicate densification, SplaTAM uses view-independent colors for Gaussians and creates a densification mask to determine whether a pixel in a new frame needs densification by considering current Gaussians and the captured depth of the new frame [S1]. For stabilizing the localization and mapping, GaussianSplattingSLAM and Gaussian-SLAM put an extra scale regularization loss on the scale of Gaussians to encourage isotropic Gaussians [S1].

#### number
On the Replica dataset, GS-SLAM achieves PSNR 31.56, SSIM 0.97, and LPIPS 0.094 [S1].

## Background
The family originates from the application of 3D Gaussian Splatting to SLAM tasks, moving beyond static novel view synthesis. GS-SLAM [180] was among the first to propose adaptive 3D Gaussian expanding and deletion based on depth and opacity [S1]. SplaTAM [181] introduced view-independent colors and densification masks to prevent duplicate creation [S1]. GaussianSplattingSLAM [182] and Gaussian-SLAM [183] introduced scale regularization for isotropic Gaussians to stabilize optimization [S1]. Other methods like LIVGaussMap [184] use LiDAR for initialization, while SGS-SLAM [185], NEDS-SLAM [186], and SemGauss-SLAM [187] extend the framework with semantic or geometric constraints [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
