# SLAM and Mapping

Also called: Simultaneous Localization and Mapping, Neural SLAM, 3DGS SLAM

## Overview
SLAM and Mapping in the context of 3D Gaussian Splatting (3DGS) solves the problem of jointly estimating camera poses and constructing a high-fidelity 3D scene representation from a stream of images, often in real-time or semi-real-time. It sets itself apart from offline reconstruction methods by integrating the differentiable rendering pipeline directly into the pose optimization loop, allowing gradients from the rendered image to update both the Gaussian parameters and the camera extrinsics [S2, unofficial]. Unlike traditional SLAM systems that rely on explicit geometric features or point clouds, modern variants often employ hybrid neural fields or explicit Gaussian primitives to maintain consistency and enable loop closure [S3].

## Shared principles
The core mechanism involves a coupled optimization loop where the scene representation $\mathcal{R}$ (a set of 3D Gaussians $\mathcal{G}$ or a neural field) and camera poses $\mathbf{T}_t$ are jointly or alternately optimized. Each Gaussian $i$ is defined by mean position $\mu_i \in \mathbb{R}^3$, covariance matrix $\Sigma_i \in \mathbb{R}^{3 \times 3}$ (parameterized by scale $\mathbf{s}_i \in \mathbb{R}^3$ and rotation $\mathbf{R}_i \in SO(3)$), opacity $\alpha_i \in [0, 1]$, and color $c_i$ [S2, unofficial].

The optimization objective minimizes a loss function $L$:
$$ L = L_{photo} + \lambda L_{depth} + L_{reg} $$
where $L_{photo} = || I_{rendered} - I_{captured} ||_1$ measures color difference, $L_{depth} = || D_{rendered} - D_{captured} ||_1$ measures depth difference (if available), and $L_{reg}$ includes regularization terms such as scale isotropy to prevent degenerate Gaussians [S2, unofficial].

Key challenges include avoiding duplicate densification in well-reconstructed areas, mitigating map drift from accumulated pose errors, and preventing Gaussian degeneracy (overly anisotropic scales) through regularization [S2, unofficial]. The explicit geometry representation provided by 3DGS enables flexible reprojection to alleviate misalignment between different viewpoints, leading to better reconstruction compared to NeRF-based methods [S1].

## Subtopics

### [Mapping Strategies](9321-mapping-strategies.md)
Mapping strategies define the correspondence mechanism used to align pixels or features between two distinct scenes or views for the purpose of alignment, editing, or fusion. In the context of radiance-field optimization, these strategies solve the problem of establishing reliable spatial matching when direct geometric priors are insufficient or when transferring attributes between scenes. This method sets itself apart from sibling optimization techniques by focusing specifically on the *correspondence* phase rather than the gradient-based parameter update, serving as a prerequisite for resampling and fusion. See [Mapping Strategies](9321-mapping-strategies.md).

### [3D Gaussian Splatting optimization](12052-3d-gaussian-splatting-optimization.md)
This family encompasses methods that jointly optimize camera poses and a 3D Gaussian scene representation from image streams, solving the problem of simultaneous localization and mapping (SLAM) or pose estimation in dynamic or blurry scenarios. It distinguishes itself from sibling methods like [Mapping Strategies](9321-mapping-strategies.md) and [Semantic-aware mapping](12053-semantic-aware-mapping.md) by explicitly handling the differentiable rendering pipeline's gradients for pose updates, and from [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md) by using explicit Gaussian primitives rather than implicit neural fields. See [3D Gaussian Splatting optimization](12052-3d-gaussian-splatting-optimization.md).

### [Semantic-aware mapping](12053-semantic-aware-mapping.md)
See [Semantic-aware mapping](12053-semantic-aware-mapping.md).

### [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md)
Neural Radiance Field (NeRF) representation is a method for implicitly modeling 3D scenes using neural networks to enable novel view synthesis. It solves the problem of reconstructing a continuous volumetric appearance field from a sparse set of 2D images by optimizing a network to predict color and density at any 3D point and viewing direction. Unlike explicit representations such as [3D Gaussian Splatting](12052-3d-gaussian-splatting.md) which store millions of discrete blobs, NeRF stores the scene implicitly within a neural network, resulting in lower memory usage but significantly slower rendering and training times. See [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md).

### CG-SLAM
**Definition**: CG-SLAM greatly improves the reconstruction quality by introducing an uncertainty map into the training process based on the rendered depth [S1].

### Combined ORB-SLAM + 3DGS
**Definition**: Combines an existing frame-based tracking system (ORB-SLAM) with 3DGS mapping. ORB-SLAM recovers poses, and 3DGS is trained with poses and sensor depth of keyframes, equivalent to offline splatting [S2, unofficial].
**Performance**: On EuRoC Machine Hall (Stereo Depth), ATE RMSE (m): ORB-SLAM3 0.052, Ours 3.190. Point-SLAM fails on all sequences [S2, unofficial].
**Comparison**: ORB-SLAM achieves best ATE, but combined approach shows no significant difference in rendering metrics compared to unified 3DGS SLAM. Unified approach is preferred for exploring 3DGS throughout the entire algorithm [S2, unofficial].

### CP-SLAM
**Definition**: A neural point-based hybrid neural field SLAM system enabling multi-agent cooperative localization and mapping with loop closure [S3].
**Steps**: Uses a keyframe-associated neural point representation inspired by PointNeRF, allowing per-point features to be adjusted during pose graph optimization. Employs a two-stage distributed-to-centralized training scheme: initial decoders are trained separately for each agent, then fused and fine-tuned jointly. Integrates odometry, loop detection, sub-map fusion, and global refinement into a unified framework [S3].

### MNE-SLAM
**Definition**: The first fully distributed multi-agent neural SLAM framework enabling accurate collaborative mapping and robust camera tracking without centralized training or raw data exchange [S3].
**Steps**: Uses a triplane+MLP hybrid neural field representation as a mapping module. Introduces an intra-to-inter loop closure strategy to reduce pose drift and align submaps across agents through peer-to-peer feature sharing and global consistency loss [S3].

### HERO-SLAM
**Definition**: A SLAM method employing a novel multiscale patch-based loss that aligns feature points, maps, and RGB-D pixels through warpings [S3].
**Steps**: Uses an INGP-like multiresolution hybrid feature grid+MLP representation for neural SDF learning [S3].

### PIN-SLAM
**Definition**: A globally consistent SLAM system using a point-based implicit neural (PIN) representation [S3].
**Steps**: Replaces grid structures with neural feature points, offering spatial flexibility and elastic correction during loop closure. Mapping alternates with odometry. Mapping step is based on a hybrid neural SDF representation with explicit neural points and implicit MLP decoders. Odometry is performed correspondence-free via second-order scan-to-map optimization. A sliding window replay buffer ensures stable incremental updates. Loop closures trigger pose graph optimization and elastic deformation of neural points, enabling consistent large-scale mapping [S3].

### Active 3D Reconstruction Algorithm
**Steps**:
1.  **Initialization**: Initialize 3D bounding box, uniformly sample goal-state candidates $S_g$ from view space, initialize DVGO coarse model $M^{(c)}$ and agent state $s_0$, append initial image to database $\{I\}$.
2.  **Global Policy**: Select next-best-view $s^*_g$ from goal candidates $S_g$ using criterion: $s^*_g = \arg\min_{s_g \in S_g} (\lambda_{cp} l_{cp}(x_{s_g}) - \lambda_{ig} E_s(s_g))$, where $x_{s_g}$ is 3D location of $s_g$.
3.  **Local Policy**: Execute planning optimization to obtain optimized trajectory $S^*$ and local next-best view $s^*$.
4.  **Action**: Move agent to $s^*$.
5.  **Observation**: Obtain new image observation $I_t$.
6.  **Update Database**: Add new observation to database $\{I\}_{i=0}^{t+1}$.
7.  **Mapping Optimization**: Optimize DVGO model $M_t$ to $M_{t+1}$ by sampling pixels from database, including synchronizing entropy volume $V(E,c)$.
8.  **Stop Check**: Stop if (1) change in entropy volume is below threshold, or (2) maximum information gain of feasible goal candidates is below threshold.
9.  **Fine Stage**: After exploration, initialize DVGO fine model $M^{(f)}$ with trained coarse model $M_T^{(c)}$. Perform stochastic training by sampling pixels from database $\{I\}_{i=0}^T$ for $N$ iterations, optimizing every $m$ iterations [S4, unofficial].

## Background
The integration of 3D Gaussian Splatting into SLAM systems emerged as a response to the need for real-time, high-quality 3D reconstruction. Traditional SLAM methods, such as ORB-SLAM, provide reliable tracking but often lack the rich appearance modeling required for photorealistic rendering [S2, unofficial]. The explicit geometry representation provided by 3DGS enables flexible reprojection to alleviate misalignment between different viewpoints, thus leading to better reconstruction compared to NeRF-based methods [S1]. The real-time rendering feature of 3DGS also makes neural-based SLAM methods more applicable for robotics tasks like relocalization, navigation, 6D pose estimation, multi-sensor calibration, and manipulation [S1].

Early approaches combined existing frame-based tracking systems with 3DGS mapping, treating the latter as an offline splatting process conditioned on poses recovered by the former [S2, unofficial]. However, unified approaches that jointly optimize poses and Gaussians have become preferred for their ability to explore the entire algorithmic pipeline within the 3DGS framework [S2, unofficial]. Concurrently, hybrid neural field approaches (e.g., CP-SLAM, PIN-SLAM) have been developed to leverage the spatial flexibility of neural points for loop closure and large-scale consistency [S3]. Active reconstruction algorithms have also been proposed to autonomously plan views that maximize information gain while reconstructing the scene [S4, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
