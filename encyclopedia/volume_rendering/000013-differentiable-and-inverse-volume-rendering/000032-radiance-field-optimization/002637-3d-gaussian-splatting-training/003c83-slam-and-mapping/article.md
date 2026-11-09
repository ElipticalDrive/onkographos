# SLAM and Mapping
Also called: 3D Gaussian SLAM, Gaussian SLAM, RGB-D SLAM with 3DGS

## Overview
This family encompasses methods that simultaneously estimate camera poses and reconstruct 3D scenes using 3D Gaussian Splatting as the explicit scene representation. Unlike standard offline training, these methods operate in a streaming or keyframe-based manner to handle dynamic environments and large-scale spaces. They distinguish themselves from sibling methods like [Physical Parameter Optimization](4111-physical-parameter-optimization.md) by focusing on geometric consistency and real-time tracking rather than physical material properties, and from [WildGaussians](6064-wildgaussians.md) by handling rigid or semi-rigid scene reconstruction rather than dynamic non-rigid deformation.

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
SplaTAM is an online RGB-D SLAM method that performs simultaneous tracking and mapping. It distinguishes itself by using view-independent colors and a specific densification mask based on depth consistency to avoid duplicate densification. It initializes Gaussians from the first frame's depth and uses depth and photometric ICP for tracking.

### [Gaussian-SLAM](9195-gaussian-slam.md)
Gaussian-SLAM is a keyframe-based SLAM method that maintains global consistency through sub-map fusion. It distinguishes itself from online streaming variants by employing a batch-optimization approach with explicit map management. It stabilizes optimization through scale regularization to encourage isotropic Gaussians and handles loop closure implicitly via fusion.

### [Mapping Strategies](9321-mapping-strategies.md)
Strategies to map pixels between two scenes for alignment.

### [Gaussian Splatting SLAM](10078-gaussian-splatting-slam.md)
A unified 3DGS-based SLAM method.

#### GS-SLAM
GS-SLAM proposes an adaptive 3D Gaussian expanding strategy to add new 3D Gaussians into the training stage and delete unreliable ones using captured depths and rendered opacity values [S1]. On the Replica dataset, GS-SLAM achieves PSNR 31.56, SSIM 0.97, and LPIPS 0.094 [S1].

#### CG-SLAM
CG-SLAM introduces an uncertainty map into the training process based on the rendered depth, which greatly improves reconstruction quality [S1].

#### SemGauss-SLAM
SemGauss-SLAM considers Gaussian’s semantic information in the simultaneous localization and mapping process by distilling 2D semantic information which can be obtained using 2D segmentation methods or provided by the dataset [S1].

#### NEDS-SLAM
NEDS-SLAM considers Gaussian’s semantic information in the simultaneous localization and mapping process by distilling 2D semantic information which can be obtained using 2D segmentation methods or provided by the dataset [S1].

#### Combined ORB-SLAM + 3DGS
This approach combines an existing frame-based tracking system (ORB-SLAM) with 3DGS mapping. ORB-SLAM recovers poses, and 3DGS is trained with poses and sensor depth of keyframes, equivalent to offline splatting [S2, unofficial]. On EuRoC Machine Hall (Stereo Depth), ATE RMSE (m) is 3.190 for the combined approach compared to 0.052 for ORB-SLAM3. While ORB-SLAM achieves best ATE, there is no significant difference in rendering metrics compared to unified 3DGS SLAM. The unified approach is preferred for exploring 3DGS throughout the entire algorithm [S2, unofficial].

## Background
3D Gaussian Splatting SLAM emerged as a response to the limitations of implicit representations (like NeRFs) in SLAM systems, which are slow to train and difficult to update online. The explicit nature of 3D Gaussians allows for real-time rendering and efficient geometric updates. Early works like GS-SLAM [S1] introduced adaptive densification, while SplaTAM [S2, unofficial] and Gaussian-SLAM [S1] established online and keyframe-based paradigms respectively. Subsequent works incorporated semantic information [S1] and hybrid tracking systems [S2, unofficial]. These methods enable downstream robotics tasks such as relocalization, navigation, and manipulation [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
