# SLAM and Mapping

Also called: SLAM, Simultaneous Localization and Mapping, Neural SLAM, Gaussian SLAM

## Overview
SLAM and Mapping in the context of 3D Gaussian Splatting (3DGS) refers to the joint estimation of camera poses and the optimization of a 3D Gaussian scene representation from a sequence of images or RGB-D frames. It solves the problem of reconstructing a consistent 3D model while tracking the camera trajectory in real-time or offline. Unlike standard 3DGS training which assumes known camera poses, SLAM methods must handle pose uncertainty, often integrating geometric constraints, depth information, or neural field representations to maintain consistency. This family distinguishes itself by coupling the differentiable rendering of Gaussians (or hybrid neural fields) with optimization loops that update both the scene geometry/appearance and the camera extrinsics simultaneously.

## Shared principles
The core mechanism of 3DGS-based SLAM involves an iterative loop that alternates between or jointly optimizes camera poses $T$ and Gaussian parameters $\mathcal{G}$ (position $\mathbf{p}$, covariance $\Sigma$, opacity $\alpha$, color $\mathbf{c}$, and spherical harmonics coefficients).

1.  **Differentiable Rendering:** The scene is rendered from the current camera pose $T$ to produce an image $\hat{I}$ and/or depth map $\hat{D}$. This is typically done via alpha-compositing of 3D Gaussians projected onto the image plane.
2.  **Loss Computation:** A loss function $\mathcal{L}$ is computed between the rendered output and the ground-truth input (image $I$, depth $D$, or semantic labels). Common terms include photometric loss ($\mathcal{L}_{rgb}$) and geometric loss ($\mathcal{L}_{depth}$).
3.  **Gradient Propagation:** Gradients $\frac{\partial \mathcal{L}}{\partial T}$ and $\frac{\partial \mathcal{L}}{\partial \mathcal{G}}$ are backpropagated through the differentiable renderer.
4.  **Optimization:** Camera poses and Gaussian parameters are updated using gradient descent (e.g., Adam) or bundle adjustment.

Key challenges addressed by specific methods include:
*   **Pose Drift:** Accumulated errors in camera trajectory estimation.
*   **Geometric Consistency:** Ensuring that the reconstructed geometry aligns with observed depth or structure.
*   **Representation Choice:** Using explicit 3D Gaussians, implicit neural fields (NeRF), or hybrid representations (neural points + MLPs) to balance rendering speed and reconstruction quality.

## Subtopics

### [CG-SLAM](9196-cg-slam.md)
CG-SLAM is a SLAM method for 3D Gaussian Splatting that integrates an uncertainty map derived from rendered depth into the training process. It addresses the reconstruction quality limitations of standard differentiable rendering pipelines by leveraging explicit geometry representation to alleviate viewpoint misalignment. It extends the standard 3DGS optimization loop by incorporating depth-based uncertainty.

*   **Mechanism:** It renders a depth map $\hat{D}$ from the current Gaussians and camera pose. An uncertainty map $U$ is computed from $\hat{D}$ (formula not specified in sources [G]). This map modulates the loss function, reducing the influence of pixels with high depth uncertainty (e.g., low texture, occlusions) during optimization.
*   **When to choose:** Use when depth information is available and geometric consistency is critical, particularly in textureless regions where standard photometric loss may fail.
*   **Link:** [CG-SLAM](9196-cg-slam.md)

### [Mapping Strategies](9321-mapping-strategies.md)
Mapping strategies define the mechanism for aligning pixels or features between two scenes or observations to facilitate spatial correspondence, pose estimation, or scene fusion. They solve the problem of establishing accurate geometric and semantic links between distinct viewpoints or temporal states in SLAM and 3D reconstruction pipelines. Unlike sibling methods that may rely on discrete feature matching or dense neural field optimization alone, these strategies range from direct pixel identity assumptions to semantic-aware correspondences and joint optimization windows.

*   **Identity Mapping:** Assumes direct pixel correspondence $(u,v) \to (u,v)$ or rigid transformation. Lacks accuracy under significant viewpoint changes.
*   **Semantic Correspondence:** Uses Vision Transformers (ViT/DINO) to extract dense semantic features $F \in \mathbb{R}^{H \times W \times C}$ and matches them via cosine similarity. Improves matching in textureless regions.
*   **Co-segmentation:** Segments images into $K$ parts using DINO clustering, then matches regions semantically before refining pixel-level correspondence. Enforces structural consistency.
*   **Joint Mapping and Pose Update:** Fuses tracked RGB-D frames into an existing map by jointly optimizing geometry, poses, and features. Uses a sliding window of $N_{kf}$ keyframes and $N_m$ sampled rays to mitigate network forgetting in online incremental learning.
*   **When to choose:** Choose Identity for simple rigid motions; Semantic/Co-segmentation for textureless or repetitive environments; Joint Mapping for online incremental reconstruction.
*   **Link:** [Mapping Strategies](9321-mapping-strategies.md)

### [Semantic-aware mapping](12053-semantic-aware-mapping.md)
Semantic-aware mapping integrates semantic information into the SLAM loop to improve robustness and consistency. It typically involves extracting semantic features (e.g., via DINO or other encoders) and using them to guide feature matching, loop closure, or regularization. This helps in handling dynamic objects and textureless regions where photometric cues are insufficient.
*   **When to choose:** Use in environments with significant semantic structure or dynamic elements where geometric consistency alone is ambiguous.
*   **Link:** [Semantic-aware mapping](12053-semantic-aware-mapping.md)

### [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md)
Neural Radiance Field (NeRF) representation is a method for modeling 3D scenes using continuous volumetric functions, typically implemented as multi-layer perceptrons (MLPs) or hybrid grid-MLP architectures, that map spatial coordinates and viewing directions to color and density. It solves the problem of novel view synthesis by learning a scene's appearance and geometry implicitly, allowing for photorealistic rendering from arbitrary camera poses. Unlike explicit representations such as [3D Gaussian Splatting](12053-3d-gaussian-splatting.md) which store millions of discrete primitives, NeRF stores the scene implicitly within network weights, offering high compression ratios at the cost of slower rendering speeds.

*   **Mechanism:** A network $f_\theta$ maps $(\mathbf{x}, \vec{\omega}) \to (\sigma, \mathbf{c})$. Rendering uses volume rendering equation: $C(\mathbf{r}) = \sum T_k \alpha_k \mathbf{c}_k$. Hybrid variants (e.g., Instant NGP) use hash grids for acceleration.
*   **When to choose:** Use when high-fidelity appearance is prioritized over real-time rendering speed, or when integrating with existing NeRF-based SLAM pipelines.
*   **Link:** [Neural Radiance Field (NeRF) Representation](12096-neural-radiance-field-nerf-representation.md)

### Combined ORB-SLAM + 3DGS
Combines an existing frame-based tracking system (ORB-SLAM) with 3DGS mapping. ORB-SLAM recovers poses, and 3DGS is trained with poses and sensor depth of keyframes, equivalent to offline splatting [S1, unofficial].

*   **Performance:** On EuRoC Machine Hall (Stereo Depth), ATE RMSE (m): ORB-SLAM3 0.052, Ours 3.190. Point-SLAM fails on all sequences [S1, unofficial].
*   **Comparison:** ORB-SLAM achieves best ATE, but combined approach shows no significant difference in rendering metrics compared to unified 3DGS SLAM. Unified approach is preferred for exploring 3DGS throughout the entire algorithm [S1, unofficial].
*   **When to choose:** Use when robust pose estimation is required from classical methods, or for offline reconstruction where real-time tracking is not needed.

### CP-SLAM
A neural point-based hybrid neural field SLAM system enabling multi-agent cooperative localization and mapping with loop closure [S2].

*   **Mechanism:** Uses a keyframe-associated neural point representation inspired by PointNeRF, allowing per-point features to be adjusted during pose graph optimization. Employs a two-stage distributed-to-centralized training scheme: initial decoders are trained separately for each agent, then fused and fine-tuned jointly. Integrates odometry, loop detection, sub-map fusion, and global refinement [S2].
*   **When to choose:** Use for multi-agent cooperative SLAM scenarios requiring loop closure and consistent map fusion.

### MNE-SLAM
The first fully distributed multi-agent neural SLAM framework enabling accurate collaborative mapping and robust camera tracking without centralized training or raw data exchange [S2].

*   **Mechanism:** Uses a triplane+MLP hybrid neural field representation as a mapping module. Introduces an intra-to-inter loop closure strategy to reduce pose drift and align submaps across agents through peer-to-peer feature sharing and global consistency loss [S2].
*   **When to choose:** Use for distributed multi-agent systems where bandwidth constraints prevent raw data exchange.

### HERO-SLAM
A SLAM method employing a novel multiscale patch-based loss that aligns feature points, maps, and RGB-D pixels through warpings [S2].

*   **Mechanism:** Uses an INGP-like multiresolution hybrid feature grid+MLP representation for neural SDF learning. Extensive evaluations show superior performance and robustness over prior implicit field-based SLAM methods, especially under challenging conditions [S2].
*   **When to choose:** Use for high-accuracy RGB-D SLAM in challenging environments requiring robust feature alignment.

### PIN-SLAM
A globally consistent SLAM system using a point-based implicit neural (PIN) representation [S2].

*   **Mechanism:** Replaces grid structures with neural feature points, offering spatial flexibility and elastic correction during loop closure. Mapping alternates with odometry. Mapping step is based on a hybrid neural SDF representation with explicit neural points and implicit MLP decoders. Odometry is performed correspondence-free via second-order scan-to-map optimization. A sliding window replay buffer ensures stable incremental updates. Loop closures trigger pose graph optimization and elastic deformation of neural points, enabling consistent large-scale mapping [S2].
*   **When to choose:** Use for large-scale mapping requiring global consistency and elastic correction during loop closure.

### Active 3D Reconstruction Algorithm
An active SLAM approach that plans camera trajectories to maximize information gain during reconstruction [S3, unofficial].

*   **Mechanism:**
    1.  **Initialization:** Initialize 3D bounding box, uniformly sample goal-state candidates $S_g$ from view space, initialize DVGO coarse model $M(c)$ and agent state $s_0$, append initial image to database $\{I\}$.
    2.  **Global Policy:** Select next-best-view $s^*_g$ from goal candidates $S_g$ using criterion: $s^*_g = \arg\min_{s_g \in S_g} (\lambda_{cp} l_{cp}(x_{s_g}) - \lambda_{ig} E_s(s_g))$, where $x_{s_g}$ is 3D location of $s_g$.
    3.  **Local Policy:** Execute planning optimization to obtain optimized trajectory $S^*$ and local next-best view $s^*$.
    4.  **Action:** Move agent to $s^*$.
    5.  **Observation:** Obtain new image observation $I_t$.
    6.  **Update Database:** Add new observation to database $\{I\}_{i=0}^{t+1}$.
    7.  **Mapping Optimization:** Optimize DVGO model $M_t$ to $M_{t+1}$ by sampling pixels from database, including synchronizing entropy volume $V(E,c)$.
    8.  **Stop Check:** Stop if (1) change in entropy volume is below threshold, or (2) maximum information gain of feasible goal candidates is below threshold.
    9.  **Fine Stage:** After exploration, initialize DVGO fine model $M(f)$ with trained coarse model $M_T(c)$. Perform stochastic training by sampling pixels from database $\{I\}_{i=0}^T$ for $N$ iterations, optimizing every $m$ iterations [S3, unofficial].
*   **When to choose:** Use for autonomous exploration tasks where maximizing reconstruction quality per unit of motion is required.

### Lightness-Decoupled Camera Tracking
A camera tracking method that decouples lightness from the image to handle lighting changes, by converting images to LAB space and discarding the L (lightness) channel during comparison [S4].

*   **Mechanism:**
    1.  Parameterize camera pose as $T = (t, q)$ with 3D translation $t$ and rotation quaternion $q$.
    2.  Initialize camera pose assuming constant camera velocity.
    3.  Convert input image $C$ and rendered image $\hat{C}$ to LAB space using $\rho$.
    4.  Discard the L channel.
    5.  Minimize $L_{tracking}$ using color (A/B channels) and depth differences.
*   **Formula:** $L_{tracking} = \lambda_{color} |\rho(\hat{C}) - \rho(C)| + \lambda_{depth} |\hat{D} - D|$, where $\rho$ is the operation of converting the image from RGB to LAB space, $C$ is the input image, $\hat{C}$ is the rendered image, $D$ is the input depth, and $\hat{D}$ is the rendered depth [S4].
*   **Parameters:** $\lambda_{color} = 0.5$, weight for the color term in the tracking loss. $\lambda_{depth} = 1.0$, weight for the depth term in the tracking loss [S4].
*   **When to choose:** Use in environments with significant lighting variations where photometric consistency is compromised.

### Active Mapping with 3D Gaussian Splatting
Extends passive mapping with active view selection to maximize information gain [S5].

*   **Mechanism:** Reconstruct environment using 3D Gaussians (e.g., via SplaTAM). Propose path candidates using frontier-based exploration. Sample cameras along the path concerning the mobility of the agent. Compute EIG for each path and choose the one with the highest EIG to continue exploration.
*   **Formula:** $P_j = \arg \max_{P_j} \text{tr} \left( \sum_{x_{acq_i} \in P_j} H''[y_{i\_acq}|x_{acq_i}, w*] \right) H''[w*|D_{train}]^{-1}$ [S5].
*   **When to choose:** Use for autonomous mapping where efficient exploration is required to minimize reconstruction time or energy.

## Background
SLAM has a long history in computer vision, traditionally relying on feature-based methods (e.g., ORB-SLAM) or direct methods (e.g., DSO, LSD-SLAM). The advent of Neural Radiance Fields (NeRF) [Mildenhall et al. 2020] introduced implicit neural representations for novel view synthesis, leading to "Neural SLAM" methods that optimize neural fields jointly with camera poses. More recently, 3D Gaussian Splatting [Kerbl et al. 2023] has been adapted for SLAM (e.g., SplaTAM, Gaussian Splatting SLAM) due to its fast rendering speed and explicit representation, offering a balance between the quality of NeRF and the efficiency of traditional point-based methods. Hybrid approaches combining neural fields and Gaussians (e.g., PIN-SLAM, CP-SLAM) are also emerging to leverage the strengths of both representations.

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] ActiveRMAP: Radiance Field for Active Mapping And Planning, https://doi.org/10.48550/arxiv.2211.12656, fetched 2026-10-26 (unofficial)
[S4] High-Fidelity SLAM Using Gaussian Splatting with Rendering-Guided Densification and Regularized Optimization, https://core.ac.uk/download/685139838.pdf, fetched 2026-10-31
[S5] FisherRF: Active View Selection and Mapping with Radiance Fields Using Fisher Information, https://doi.org/10.1007/978-3-031-72624-8_24, fetched 2026-10-31
