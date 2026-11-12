# SLAM and Mapping
Also called: 3DGS SLAM, Gaussian Splatting SLAM, Neural SLAM with 3DGS

## Overview
This family encompasses methods that integrate Simultaneous Localization and Mapping (SLAM) with 3D Gaussian Splatting (3DGS) to jointly estimate camera poses and reconstruct a scene representation in real-time or near-real-time. It solves the problem of maintaining a consistent, high-fidelity 3D model while tracking a moving camera, leveraging the differentiable rasterization of 3D Gaussians for both rendering and optimization. It is distinguished from sibling methods like [Mapping Strategies](9321-mapping-strategies.md) by its focus on temporal consistency and pose estimation rather than cross-scene attribute transfer, and from [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md) by its use of explicit Gaussian primitives which enable faster rendering and more flexible geometric manipulation [S1].

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
Mapping strategies define the correspondence mechanism used to align pixels or features between two distinct scene representations, typically for tasks such as 3D editing, style transfer, or multi-scene fusion. Unlike standard SLAM mapping which aligns a current frame to a static map, these strategies map between two complete or partial scene states. They are distinguished from sibling methods like [Semantic-aware mapping](12053-semantic-aware-mapping.md) by their focus on cross-scene attribute transfer rather than intra-scene semantic consistency, and from [3D Gaussian Splatting optimization](12052-3d-gaussian-splatting-optimization.md) by their reliance on feature-space or segmentation-based correspondence rather than direct geometric rasterization gradients.

### [3D Gaussian Splatting optimization](12052-3d-gaussian-splatting-optimization.md)
This family encompasses methods that optimize a scene representation $\mathcal{R}$ composed of 3D Gaussians $\mathcal{G}$ and camera poses $\mathbf{T}_t$ to minimize the difference between rendered outputs and captured sensor data. It distinguishes itself from sibling methods like [Mapping Strategies](9321-mapping-strategies.md) and [Semantic-aware mapping](12053-semantic-aware-mapping.md) by explicitly leveraging the differentiable rasterization of 3D Gaussians for simultaneous localization and mapping (SLAM) or pose estimation, often utilizing specific densification, regularization, or keyframe management strategies to handle the dynamic nature of the Gaussian set.

### [Semantic-aware mapping](12053-semantic-aware-mapping.md)
Not given in the sources.

### [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md)
Not given in the sources.

### CG-SLAM
#### definition
CG-SLAM greatly improves the reconstruction quality [S1].
#### step
Introduce an uncertainty map into the training process based on the rendered depth [S1].

### Combined ORB-SLAM + 3DGS
#### definition
Combines an existing frame-based tracking system (ORB-SLAM) with 3DGS mapping. ORB-SLAM recovers poses, and 3DGS is trained with poses and sensor depth of keyframes, equivalent to offline splatting [S2, unofficial].
#### number
On EuRoC Machine Hall (Stereo Depth), ATE RMSE (m): ORB-SLAM3 0.052, Ours 3.190. Point-SLAM fails on all sequences [S2, unofficial].
#### comparison
ORB-SLAM achieves best ATE, but combined approach shows no significant difference in rendering metrics compared to unified 3DGS SLAM. Unified approach is preferred for exploring 3DGS throughout the entire algorithm [S2, unofficial].

### CP-SLAM
#### definition
A neural point-based hybrid neural field SLAM system enabling multi-agent cooperative localization and mapping with loop closure [S3].
#### step
Uses a keyframe-associated neural point representation inspired by PointNeRF, allowing per-point features to be adjusted during pose graph optimization [S3].
Employs a two-stage distributed-to-centralized training scheme: initial decoders are trained separately for each agent, then fused and fine-tuned jointly [S3].
Integrates odometry, loop detection, sub-map fusion, and global refinement into a unified framework [S3].

### MNE-SLAM
#### definition
The first fully distributed multi-agent neural SLAM framework enabling accurate collaborative mapping and robust camera tracking without centralized training or raw data exchange [S3].
#### step
Uses a triplane+MLP hybrid neural field representation as a mapping module [S3].
Introduces an intra-to-inter loop closure strategy to reduce pose drift and align submaps across agents through peer-to-peer feature sharing and global consistency loss [S3].

### HERO-SLAM
#### definition
A SLAM method employing a novel multiscale patch-based loss that aligns feature points, maps, and RGB-D pixels through warpings [S3].
#### step
Uses an INGP-like multiresolution hybrid feature grid+MLP representation for neural SDF learning [S3].

### PIN-SLAM
#### definition
A globally consistent SLAM system using a point-based implicit neural (PIN) representation [S3].
#### step
Replaces grid structures with neural feature points, offering spatial flexibility and elastic correction during loop closure [S3].
Mapping alternates with odometry [S3].
Mapping step is based on a hybrid neural SDF representation with explicit neural points and implicit MLP decoders [S3].
Odometry is performed correspondence-free via second-order scan-to-map optimization [S3].
A sliding window replay buffer ensures stable incremental updates [S3].
Loop closures trigger pose graph optimization and elastic deformation of neural points, enabling consistent large-scale mapping [S3].

### Active 3D Reconstruction Algorithm
#### step
Initialization: Initialize 3D bounding box, uniformly sample goal-state candidates Sg from view space, initialize DVGO coarse model M(c) and agent state s0, append initial image to database {I} [S4, unofficial].
Global Policy: Select next-best-view s*g from goal candidates Sg using criterion: s*g = argmin_{sg ∈ Sg} (λ_cp l_cp(x_sg) - λ_ig E_s(sg)), where x_sg is 3D location of sg [S4, unofficial].
Local Policy: Execute planning optimization (Sec 4.2) to obtain optimized trajectory S* and local next-best view s* [S4, unofficial].
Action: Move agent to s* [S4, unofficial].
Observation: Obtain new image observation I_t [S4, unofficial].
Update Database: Add new observation to database {I}_{i=0}^{t+1} [S4, unofficial].
Mapping Optimization: Optimize DVGO model M_t to M_{t+1} by sampling pixels from database, including synchronizing entropy volume V(E,c) [S4, unofficial].
Stop Check: Stop if (1) change in entropy volume is below threshold, or (2) maximum information gain of feasible goal candidates is below threshold [S4, unofficial].
Fine Stage: After exploration, initialize DVGO fine model M(f) with trained coarse model M_T(c). Perform stochastic training by sampling pixels from database {I}_i=0^T for N iterations, optimizing every m iterations [S4, unofficial].

## Background
The integration of 3D Gaussian Splatting into SLAM systems emerged as a response to the computational bottlenecks of NeRF-based SLAM methods. While NeRFs offer high-quality rendering, their slow training and inference speeds limit their applicability in real-time SLAM [S1]. 3DGS provides explicit geometry representation, enabling flexible reprojection to alleviate misalignment between different viewpoints, thus leading to better reconstruction compared to NeRF-based methods [S1]. The real-time rendering feature of 3DGS also makes neural-based SLAM methods more applicable for robotics tasks such as relocalization, navigation, 6D pose estimation, multi-sensor calibration, and manipulation [S1].

Early approaches combined classical SLAM systems (like ORB-SLAM) with 3DGS mapping, treating them as separate modules where ORB-SLAM provides poses and 3DGS handles the mapping [S2, unofficial]. Unified approaches, however, optimize both poses and Gaussians jointly, leveraging the differentiable nature of 3DGS for end-to-end training [S2, unofficial]. Recent developments have extended these methods to multi-agent scenarios (e.g., CP-SLAM, MNE-SLAM) and active reconstruction (e.g., Active 3D Reconstruction Algorithm), demonstrating the versatility of Gaussian-based representations in complex SLAM tasks [S3][S4, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S3] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S4] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
