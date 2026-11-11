# SLAM and Mapping
Also called: 3D Gaussian Splatting SLAM, Gaussian SLAM, Neural Field SLAM (for hybrid variants)
## Overview
This family encompasses methods that simultaneously estimate camera poses and reconstruct a scene representation using 3D Gaussians or hybrid neural fields from streaming sensor data. Unlike batch optimization [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md), these methods operate in a continuous loop, integrating tracking and mapping to handle dynamic environments and large-scale scenes. They are distinguished by their ability to maintain map consistency, handle loop closures, and often leverage the explicit geometry or real-time rendering capabilities of 3D Gaussians to alleviate misalignment issues common in NeRF-based methods [S1].
## Shared principles
All methods in this family share the core objective of optimizing a scene representation $\mathcal{R}$ (composed of 3D Gaussians $\mathcal{G}$ or hybrid neural fields) and camera poses $\mathbf{T}_t$ to minimize the difference between rendered outputs and captured sensor data.

**Scene Representation:**
The scene is typically represented as a collection of 3D Gaussians. Each Gaussian $i$ is defined by:
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

### [Mapping Strategies](9321-mapping-strategies.md)
Mapping strategies define the correspondence between pixels or features of two distinct scenes (e.g., a source and a target scene) to enable operations such as 3D editing, style transfer, or multi-scene fusion. Unlike sibling methods that focus on SLAM tracking or densification, these strategies address the problem of spatial alignment when transferring attributes (density, radiance, or geometry) between scenes that may differ in viewpoint, content, or structure. They are chosen when the goal is to map visual or geometric properties from one radiance field to another, rather than estimating camera poses or building a map from scratch. See [Mapping Strategies](9321-mapping-strategies.md).

### [3D Gaussian Splatting optimization](12052-3d-gaussian-splatting-optimization.md)
This family of methods optimizes a scene representation composed of 3D Gaussians and camera poses to minimize the difference between rendered outputs and captured sensor data, enabling simultaneous localization and mapping (SLAM) [Parent](9268-slam-and-mapping.md). Unlike batch-only optimization, these methods integrate tracking and mapping in a streaming or keyframe-based loop, distinguishing themselves by specific strategies to handle densification, map consistency, and Gaussian degeneracy. See [3D Gaussian Splatting optimization](12052-3d-gaussian-splatting-optimization.md).

### [Semantic-aware mapping](12053-semantic-aware-mapping.md)
See [Semantic-aware mapping](12053-semantic-aware-mapping.md).

### [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md)
See [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md).

### CG-SLAM
**Definition**
CG-SLAM introduces an uncertainty map into the training process based on the rendered depth, which greatly improves the reconstruction quality [S1].

### Combined ORB-SLAM + 3DGS
**Definition**
This approach combines an existing frame-based tracking system (ORB-SLAM) with 3DGS mapping. ORB-SLAM recovers poses, and 3DGS is trained with poses and sensor depth of keyframes, equivalent to offline splatting [S2, unofficial].

**Comparison**
On EuRoC Machine Hall (Stereo Depth), the ATE RMSE (m) for ORB-SLAM3 is 0.052, while the combined approach yields 3.190. Point-SLAM fails on all sequences [S2, unofficial]. While ORB-SLAM achieves the best ATE, the combined approach shows no significant difference in rendering metrics compared to unified 3DGS SLAM. The unified approach is preferred for exploring 3DGS throughout the entire algorithm [S2, unofficial].

### CP-SLAM
**Definition**
CP-SLAM is a neural point-based hybrid neural field SLAM system enabling multi-agent cooperative localization and mapping with loop closure [S3].

**Steps**
*   Uses a keyframe-associated neural point representation inspired by PointNeRF, allowing per-point features to be adjusted during pose graph optimization [S3].
*   Employs a two-stage distributed-to-centralized training scheme: initial decoders are trained separately for each agent, then fused and fine-tuned jointly [S3].
*   Integrates odometry, loop detection, sub-map fusion, and global refinement into a unified framework [S3].

### MNE-SLAM
**Definition**
MNE-SLAM is the first fully distributed multi-agent neural SLAM framework enabling accurate collaborative mapping and robust camera tracking without centralized training or raw data exchange [S3].

**Steps**
*   Uses a triplane+MLP hybrid neural field representation as a mapping module [S3].
*   Introduces an intra-to-inter loop closure strategy to reduce pose drift and align submaps across agents through peer-to-peer feature sharing and global consistency loss [S3].

### HERO-SLAM
**Definition**
HERO-SLAM is a SLAM method employing a novel multiscale patch-based loss that aligns feature points, maps, and RGB-D pixels through warpings [S3].

**Steps**
*   Uses an INGP-like multiresolution hybrid feature grid+MLP representation for neural SDF learning [S3].

### PIN-SLAM
**Definition**
PIN-SLAM is a globally consistent SLAM system using a point-based implicit neural (PIN) representation [S3].

**Steps**
*   Replaces grid structures with neural feature points, offering spatial flexibility and elastic correction during loop closure [S3].
*   Mapping alternates with odometry [S3].
*   Mapping step is based on a hybrid neural SDF representation with explicit neural points and implicit MLP decoders [S3].
*   Odometry is performed correspondence-free via second-order scan-to-map optimization [S3].
*   A sliding window replay buffer ensures stable incremental updates [S3].
*   Loop closures trigger pose graph optimization and elastic deformation of neural points, enabling consistent large-scale mapping [S3].

### Active 3D Reconstruction Algorithm
**Steps**
1.  **Initialization:** Initialize 3D bounding box, uniformly sample goal-state candidates $S_g$ from view space, initialize DVGO coarse model $M^{(c)}$ and agent state $s_0$, append initial image to database $\{I\}$ [S4, unofficial].
2.  **Global Policy:** Select next-best-view $s^*_g$ from goal candidates $S_g$ using criterion: $s^*_g = \arg\min_{s_g \in S_g} (\lambda_{cp} l_{cp}(x_{s_g}) - \lambda_{ig} E_s(s_g))$, where $x_{s_g}$ is 3D location of $s_g$ [S4, unofficial].
3.  **Local Policy:** Execute planning optimization to obtain optimized trajectory $S^*$ and local next-best view $s^*$ [S4, unofficial].
4.  **Action:** Move agent to $s^*$ [S4, unofficial].
5.  **Observation:** Obtain new image observation $I_t$ [S4, unofficial].
6.  **Update Database:** Add new observation to database $\{I\}_{i=0}^{t+1}$ [S4, unofficial].
7.  **Mapping Optimization:** Optimize DVGO model $M_t$ to $M_{t+1}$ by sampling pixels from database, including synchronizing entropy volume $V(E,c)$ [S4, unofficial].
8.  **Stop Check:** Stop if (1) change in entropy volume is below threshold, or (2) maximum information gain of feasible goal candidates is below threshold [S4, unofficial].
9.  **Fine Stage:** After exploration, initialize DVGO fine model $M^{(f)}$ with trained coarse model $M_T^{(c)}$. Perform stochastic training by sampling pixels from database $\{I\}_{i=0}^T$ for $N$ iterations, optimizing every $m$ iterations [S4, unofficial].

## Background
The integration of SLAM with 3D Gaussian Splatting emerged as a response to the limitations of NeRF-based SLAM, particularly regarding rendering speed and geometric explicitness. 3DGS provides explicit geometry representation that enables flexible reprojection to alleviate misalignment between different viewpoints, leading to better reconstruction compared to NeRF-based methods [S1]. The real-time rendering feature of 3DGS also makes neural-based SLAM methods more applicable for robotics tasks like relocalization, navigation, 6D pose estimation, multi-sensor calibration, and manipulation [S1]. Early unified approaches, such as combining ORB-SLAM with 3DGS, demonstrated that while classical SLAM might offer superior pose accuracy (ATE), the rendering quality of unified 3DGS SLAM is competitive, motivating fully integrated frameworks [S2, unofficial]. Hybrid neural field approaches (e.g., CP-SLAM, PIN-SLAM) further expanded the family by incorporating neural points and implicit fields to handle multi-agent scenarios and large-scale consistency [S3]. Active reconstruction algorithms using radiance fields (e.g., ActiveRMAP) have also been adapted to guide camera motion for efficient mapping [S4, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
