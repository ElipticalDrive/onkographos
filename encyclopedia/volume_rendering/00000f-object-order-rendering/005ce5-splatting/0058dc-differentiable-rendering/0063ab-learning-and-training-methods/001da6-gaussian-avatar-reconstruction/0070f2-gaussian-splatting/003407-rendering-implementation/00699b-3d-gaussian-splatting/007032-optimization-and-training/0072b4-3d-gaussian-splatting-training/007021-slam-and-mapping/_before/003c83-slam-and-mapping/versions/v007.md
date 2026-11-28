# SLAM and Mapping

Also called: 3D Gaussian Splatting SLAM, Neural SLAM, Active 3D Reconstruction

## Overview
SLAM and Mapping in the context of 3D Gaussian Splatting (3DGS) solves the problem of simultaneously estimating camera poses and constructing a high-fidelity, real-time renderable scene representation from image streams. It distinguishes itself from sibling optimization techniques by explicitly handling the differentiable rendering pipeline's gradients for pose updates, leveraging the explicit Gaussian primitives for faster convergence and better geometry than implicit NeRF-based methods, while avoiding the static assumptions of traditional Structure-from-Motion (SfM) pipelines.

## Shared principles
The core mechanism involves a coupled optimization loop where the scene representation $\mathcal{R}$ (a set of 3D Gaussians $\mathcal{G}$) and camera poses $\mathbf{T}_t$ are jointly or alternately optimized. Each Gaussian $i$ is defined by mean position $\mu_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (parameterized by scale $\mathbf{s}_i \in \mathbb{R}^3$ and rotation $\mathbf{R}_i \in SO(3)$), opacity $\alpha_i \in [0, 1]$, and color $c_i$ (often stored as spherical harmonics coefficients).

The optimization objective minimizes a loss function $L$:
$$ L = L_{photo} + \lambda L_{depth} + L_{reg} $$
where $L_{photo} = || I_{rendered} - I_{captured} ||_1$ measures color difference, $L_{depth} = || D_{rendered} - D_{captured} ||_1$ measures depth difference (if available), and $L_{reg}$ includes regularization terms such as scale isotropy to prevent degenerate Gaussians.

Key challenges include avoiding duplicate densification in well-reconstructed areas, mitigating map drift from accumulated pose errors, and preventing Gaussian degeneracy (overly anisotropic scales) through regularization. The rendering pipeline uses a differentiable rasterizer that projects 3D Gaussians to 2D screen space, sorts them by depth, and blends them front-to-back, allowing gradients to flow back to both the Gaussian parameters and the camera pose.

## Subtopics

### [Mapping Strategies](9321-mapping-strategies.md)
Mapping strategies define the correspondence mechanism used to align pixels or features between two distinct scenes or views for the purpose of alignment, editing, or fusion. In the context of radiance-field optimization, these strategies solve the problem of establishing reliable spatial matching when direct geometric priors are insufficient or when transferring attributes between scenes. This method sets itself apart from sibling optimization techniques by focusing specifically on the *correspondence* phase rather than the gradient-based parameter update, serving as a prerequisite for resampling and fusion. See [Mapping Strategies](9321-mapping-strategies.md).

### [3D Gaussian Splatting optimization](12052-3d-gaussian-splatting-optimization.md)
This family encompasses methods that jointly optimize camera poses and a 3D Gaussian scene representation from image streams, solving the problem of simultaneous localization and mapping (SLAM) or pose estimation in dynamic or blurry scenarios. It distinguishes itself from sibling methods like [Mapping Strategies](9321-mapping-strategies.md) and [Semantic-aware mapping](12053-semantic-aware-mapping.md) by explicitly handling the differentiable rendering pipeline's gradients for pose updates, and from [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md) by using explicit Gaussian primitives rather than implicit neural fields. See [3D Gaussian Splatting optimization](12052-3d-gaussian-splatting-optimization.md).

### [Semantic-aware mapping](12053-semantic-aware-mapping.md)
See [Semantic-aware mapping](12053-semantic-aware-mapping.md).

### [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md)
See [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md).

### CG-SLAM
**Definition**
CG-SLAM greatly improves reconstruction quality by introducing an uncertainty map into the training process based on the rendered depth [S1].

### Combined ORB-SLAM + 3DGS
**Definition**
This approach combines an existing frame-based tracking system (ORB-SLAM) with 3DGS mapping. ORB-SLAM recovers poses, and 3DGS is trained with poses and sensor depth of keyframes, equivalent to offline splatting [S2, unofficial].

**Comparison**
On EuRoC Machine Hall (Stereo Depth), ATE RMSE (m): ORB-SLAM3 0.052, Ours 3.190. Point-SLAM fails on all sequences [S2, unofficial]. ORB-SLAM achieves best ATE, but the combined approach shows no significant difference in rendering metrics compared to unified 3DGS SLAM. The unified approach is preferred for exploring 3DGS throughout the entire algorithm [S2, unofficial].

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
*   **Initialization:** Initialize 3D bounding box, uniformly sample goal-state candidates $S_g$ from view space, initialize DVGO coarse model $M(c)$ and agent state $s_0$, append initial image to database $\{I\}$ [S4, unofficial].
*   **Global Policy:** Select next-best-view $s^*_g$ from goal candidates $S_g$ using criterion: $s^*_g = \arg\min_{s_g \in S_g} (\lambda_{cp} l_{cp}(x_{s_g}) - \lambda_{ig} E_s(s_g))$, where $x_{s_g}$ is 3D location of $s_g$ [S4, unofficial].
*   **Local Policy:** Execute planning optimization to obtain optimized trajectory $S^*$ and local next-best view $s^*$ [S4, unofficial].
*   **Action:** Move agent to $s^*$ [S4, unofficial].
*   **Observation:** Obtain new image observation $I_t$ [S4, unofficial].
*   **Update Database:** Add new observation to database $\{I\}_{i=0}^{t+1}$ [S4, unofficial].
*   **Mapping Optimization:** Optimize DVGO model $M_t$ to $M_{t+1}$ by sampling pixels from database, including synchronizing entropy volume $V(E,c)$ [S4, unofficial].
*   **Stop Check:** Stop if (1) change in entropy volume is below threshold, or (2) maximum information gain of feasible goal candidates is below threshold [S4, unofficial].
*   **Fine Stage:** After exploration, initialize DVGO fine model $M(f)$ with trained coarse model $M_T(c)$. Perform stochastic training by sampling pixels from database $\{I\}_{i=0}^T$ for $N$ iterations, optimizing every $m$ iterations [S4, unofficial].

## Background
The integration of SLAM with 3D Gaussian Splatting emerged as a response to the limitations of implicit neural representations (NeRFs) in real-time applications. While NeRFs offer high-quality rendering, their training times and lack of explicit geometry hindered their use in dynamic SLAM systems. 3DGS, introduced by Kerbl et al. [2023], provided an explicit, differentiable, and real-time renderable representation. Early works like [S2, unofficial] demonstrated that combining traditional SLAM (e.g., ORB-SLAM) with 3DGS could leverage reliable pose estimation while achieving high-fidelity mapping. Subsequent research expanded this into fully unified frameworks where poses and Gaussians are optimized jointly, and into hybrid neural fields (e.g., CP-SLAM, PIN-SLAM) that combine the benefits of explicit points with implicit neural decoding for better generalization and loop closure handling [S3]. Active reconstruction methods further extended this by using the radiance field for view planning [S4, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
