# Combined ORB-SLAM + 3DGS

Also called: ORB-SLAM + 3DGS, Offline Splatting with ORB-SLAM

## Overview
This method combines a mature frame-based SLAM system (ORB-SLAM) with 3D Gaussian Splatting (3DGS) to achieve simultaneous localization and mapping. It solves the problem of integrating explicit geometric tracking with differentiable rendering by decoupling pose estimation from map optimization. Unlike unified 3DGS-based SLAM methods where tracking and mapping are tightly coupled, this approach uses ORB-SLAM for reliable pose recovery and treats 3DGS training as an offline splatting process using the recovered poses and sensor depth.

## How it works
The method operates in two distinct stages: tracking and mapping.

**Inputs:**
- A sequence of RGB-D images (or stereo images with depth estimation).
- Initial camera intrinsics.

**Outputs:**
- Camera poses for all frames.
- A set of 3D Gaussians representing the scene.

**Data Structures:**
1.  **ORB-SLAM Map:** Contains keyframes, map points, and camera poses. The map points are triangulated from feature matches.
2.  **3DGS Representation:** A list of Gaussians, each defined by:
    -   Mean position $\mu \in \mathbb{R}^3$
    -   Covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ (or rotation $q \in \mathbb{R}^4$ and scale $s \in \mathbb{R}^3$)
    -   Opacity $\alpha \in [0, 1]$
    -   Spherical Harmonics coefficients $c_{lm}$ (optional, depending on implementation)

**Steps:**

1.  **Pose Estimation (ORB-SLAM):**
    Run the ORB-SLAM pipeline (e.g., ORB-SLAM2 or ORB-SLAM3) on the input video stream.
    -   Extract ORB features from each frame.
    -   Perform feature matching between consecutive frames and with the local map.
    -   Estimate camera pose $T_{cw}$ using PnP and bundle adjustment.
    -   Identify keyframes and update the map.
    -   *Note:* This step provides robust tracking and loop closure, which are often challenging for pure 3DGS-based trackers.

2.  **Data Preparation for 3DGS:**
    For each keyframe $k$ with known pose $T_{cw}^{(k)}$ and associated depth map $D^{(k)}$:
    -   Extract 2D pixels $(u, v)$ from the depth map.
    -   Back-project pixels to 3D space using camera intrinsics $K$:
        $$
        \mathbf{X}_{3D} = K^{-1} \begin{bmatrix} u \\ v \\ 1 \end{bmatrix} \cdot D^{(k)}_{u,v}
        $$
    -   Transform 3D points to the world frame:
        $$
        \mathbf{X}_{world} = T_{cw}^{(k)} \begin{bmatrix} \mathbf{X}_{3D} \\ 1 \end{bmatrix}
        $$
    -   These points serve as the initial positions for the 3D Gaussians.

3.  **3DGS Initialization:**
    -   Initialize a Gaussian at each back-projected point $\mathbf{X}_{world}$.
    -   Set initial covariance $\Sigma$ based on the local surface normal and depth uncertainty.
    -   Set initial opacity $\alpha$ and color (from the RGB image).
    -   If using Spherical Harmonics (SH), initialize coefficients based on the view-dependent color.

4.  **Offline Training (Splatting):**
    Optimize the 3DGS parameters (positions, covariances, opacities, colors/SH) using the fixed poses from ORB-SLAM.
    -   Render the scene from the known camera poses.
    -   Compute the photometric loss between rendered images and ground truth RGB images.
    -   Update Gaussian parameters via gradient descent.
    -   *Note:* Poses are fixed during this stage. This is equivalent to "offline splatting."

**Parameters:**
-   Learning rate for 3DGS optimization: Not given in the sources.
-   Number of training iterations: Not given in the sources.
-   SH degree: Optional. Omitted in synthetic datasets without view-dependent effects (e.g., Replica).

**Edge Cases and Pitfalls:**
-   **Depth Noise:** ORB-SLAM relies on feature matching; if features are sparse, depth estimation for 3DGS initialization may be noisy.
-   **Loop Closure:** While ORB-SLAM handles loop closure, the 3DGS map must be consistent across loops. The sources do not specify how loop closure corrections in ORB-SLAM are propagated to the 3DGS map.
-   **Pose Accuracy:** ORB-SLAM provides high ATE accuracy, but errors in pose can lead to artifacts in the 3DGS map.

## Subtopics

### [CG-SLAM](9196-cg-slam.md)
Combines geometric and appearance-based tracking. Choose when appearance changes are significant.

### [CP-SLAM](11948-cp-slam.md)
Color and pose SLAM. Choose when color consistency is critical for tracking.

### [iMAP](11979-imap.md)
Implicit mapping with adaptive sampling. Choose for dense, high-quality maps without explicit primitives.

### [Semantic-aware mapping](12053-semantic-aware-mapping.md)
Uses semantic labels for mapping. Choose when object-level understanding is needed.

### [RNR-Map Construction](16526-rnr-map-construction.md)
Neural rendering for map construction. Choose for neural implicit representations.

### [Hierarchical Submap Alignment](21123-hierarchical-submap-alignment.md)
Aligns submaps for large-scale scenes. Choose for very large environments.

### [Neural Implicit Representation](22462-neural-implicit-representation.md)
Uses neural networks for scene representation. Choose for smooth, continuous representations.

## Operating characteristics
-   **Time Complexity:** Tracking is dominated by ORB-SLAM's feature extraction and matching. Mapping is dominated by 3DGS optimization, which is iterative. Total time is the sum of tracking time and offline training time.
-   **Memory Complexity:** Depends on the number of Gaussians. The sources report memory usage for RGB-D SLAM:
    -   Point-SLAM: 3.97 MB (w/o SH), 11.47 MB (w. SH)
    -   ORB+GS: 38.0 MB (w/o SH), 45.97 MB (w. SH)
    -   Another variant: 186.5 MB (Not specified if w/o or w. SH)
-   **Performance:**
    -   **ATE RMSE (EuRoC Machine Hall with Stereo Depth):**
        -   ORB-SLAM3: 0.052 m
        -   Combined ORB-SLAM + 3DGS: 3.190 m
        -   Point-SLAM: Fails on all sequences
        -   Vins-Fusion: 0.500 m
        -   SVO: 0.120 m
    -   **Rendering Metrics (TUM and Replica RGBD):**
        -   ORB+GS (w. SH) PSNR: 25.44 (TUM), Not given (Replica)
        -   ORB+GS (w/o SH) PSNR: 25.12 (TUM), 37.11 (Replica)
        -   Ours (Unified 3DGS) PSNR: 21.89 (TUM), 38.94 (Replica)
        -   No significant difference in rendering metrics between combined and unified approaches.
-   **Strengths:** Robust tracking from ORB-SLAM; high-quality rendering from 3DGS.
-   **Limitations:** Lower ATE accuracy compared to pure ORB-SLAM; offline training requires all poses to be known.
-   **When to use:** When robust tracking is more important than real-time mapping, or when leveraging existing ORB-SLAM infrastructure.
-   **When not to use:** When real-time mapping is required, or when unified optimization is preferred for end-to-end consistency.

## Background
The method was proposed in "Gaussian Splatting SLAM" [S1, unofficial] as a comparison baseline to a unified 3DGS-based SLAM method. It leverages the maturity of ORB-SLAM for tracking and the efficiency of 3DGS for rendering. The approach is also discussed in "Advancement in Neural Radiance Fields Propulsion for SLAM: A Novel Upgrade" [S2], which mentions using ORB-SLAM2's frontend with a voxel grid backend for implicit maps, though this is slightly different from the explicit 3DGS approach.

## Sources
[S1] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
[S2] Advancement in Neural Radiance Fields Propulsion for SLAM: A Novel Upgrade, https://www.atlantis-press.com/proceedings/icaic-24/126003439, fetched 2026-10-28
