# Neural Implicit SLAM

Also called: NeRF-SLAM, Implicit Neural SLAM, Neural Field SLAM

## Overview
Neural Implicit SLAM is a simultaneous localization and mapping technique that constructs a continuous 3D scene representation using a neural network $F$ mapping spatial coordinates to scene properties, while simultaneously optimizing camera poses $T_t$ [S1, S3]. It differs from explicit grid-based SLAM (e.g., [Dense Grid SLAM](12345-dense-grid-slam.md)) by offering resolution-independent geometry and from traditional feature-based SLAM (e.g., [ORB-SLAM](12340-orb-slam.md)) by leveraging dense photometric and geometric constraints for tracking [S3]. Unlike [Gaussian Splatting SLAM](12350-gaussian-splatting-slam.md), which uses explicit parametric primitives, this family relies on implicit neural fields, allowing for smoother surface representations and easier integration of semantic information, though often at higher computational cost during initialization [S1].

## Shared principles
The family relies on a neural network $F$ that maps spatial coordinates $\mathbf{x}$ to scene properties [G]. For appearance, $F$ outputs color $\mathbf{c}$ and volume density $\sigma$ (or occupancy) [S1]. For geometry, $F$ may output a Signed Distance Function (SDF) value [S1]. Rendering is performed via volume rendering or ray marching: points are sampled along camera rays, properties are predicted by $F$, and results are aggregated using alpha compositing or weighted sums [S1]. Optimization jointly updates the network weights and camera poses $T_t$ to minimize photometric ($\ell_{rgb}$) and geometric ($\ell_{depth}$ or SDF) losses [S1].

## Subtopics

### [Hybrid Implicit Neural Representation SLAM Sampling & Optimization](23458-hybrid-implicit-neural-representation-slam-sampling-optimiza.md)
This method implements a real-time RGB-D SLAM framework that optimizes an implicit neural representation using hybrid spatial encodings (combining dense or hash grids with tri-planes) and a composite loss function tailored for tracking and mapping. It solves the problem of efficient, high-accuracy scene reconstruction and pose estimation by jointly optimizing the neural implicit function and camera poses using cached sampled rays. Unlike pure neural implicit methods which may suffer from high memory usage or slow convergence, this approach uses explicit hybrid encoding to accelerate feature lookup while maintaining the continuous representation benefits of neural fields. See [Hybrid Implicit Neural Representation SLAM Sampling & Optimization](23458-hybrid-implicit-neural-representation-slam-sampling-optimiza.md) for implementation details.

### SLAIM
SLAIM is a SLAM method that introduces a coarse-to-fine tracking pipeline and improves photometric bundle adjustment for NeRF-based mapping [S1]. It maintains NeRF’s original volume density formulation while introducing a KL regularization over the ray termination distribution [S1]. This addresses the challenge of high-frequency renderings in NeRF hindering image alignment [S1].

### SNI-SLAM
SNI-SLAM is a dense NeRF-based RGB-D SLAM system for accurate real-time 3D semantic mapping [S1]. It introduces a hierarchical semantic encoding and cross-attention mechanism to enable mutual reinforcement between appearance, geometry, and semantics modalities [S1]. It further proposes a novel one-way decoder design to enhance inter-modal information flow without reverse interference [S1].

### KN-SLAM
KN-SLAM is a NeRF-based SLAM method that integrates local feature correspondences for coarse pose initialization and joint optimization [S1]. It uses global image features and local matches for explicit loop detection [S1]. It performs pose graph optimization and global refinement of the neural map to ensure consistency [S1].

### VI-NeRF-SLAM
VI-NeRF-SLAM merges traditional SLAM method benefits with NeRF mapping to facilitate immediate, dense 3D scene reconstruction online without prior training [S2]. It estimates pixel depth and pose based on constraints in reprojecting feature points [S2]. It refines the depiction of 3D scenes through the use of ray integration and networks [S2]. It includes a dynamic error correction technique for IMUs [S2]. It includes a method for managing closed loops in NeRF mapping [S2]. It surpasses current benchmark techniques in reconstruction and tracking precision on Euroc and Replica datasets, particularly in scenarios limited to grayscale imagery [S2].

### NeRF-SLAM Problem Formulation
NeRF-SLAM Problem Formulation defines the task as a continuous online learning problem where a stream of synchronized RGB-D input frames is processed to optimize a neural implicit function $F$ and camera pose $T_t$ using cached sampled rays [S3]. For each pixel $\mathbf{x}$ in an RGB-D frame, $N$ points are sampled along the camera ray, each at distance $d_i$ with position $\mathbf{p}_i$ [S3]. Appearance $\mathbf{c}_i$ and geometric properties $g_i$ are predicted for each sample point $\mathbf{p}_i$ using the learnable neural implicit function $F$ [S3]:
$$ (\mathbf{c}_i, g_i) = F(\mathbf{p}_i) $$
The weight $w_i$ for each sample is determined by a geometric rendering function $G$ applied to the geometric properties:
$$ w_i = G(g_i) $$
Rendered color and depth are computed as weighted sums:
$$ \tilde{\mathbf{c}}_{\mathbf{x}} = \sum_{i=1}^N w_i \cdot \mathbf{c}_i, \quad \tilde{d}_{\mathbf{x}} = \sum_{i=1}^N w_i \cdot d_i $$
Optimization minimizes photometric and geometric constraints: $e^p_{\mathbf{x}} = \tilde{\mathbf{c}}_{\mathbf{x}} - \mathbf{c}_{\mathbf{x}}$ and $e^g_{\mathbf{x}} = \tilde{d}_{\mathbf{x}} - d_{\mathbf{x}}$ [S3]. The weight is formulated as $w_i = G(F_{geo}(\mathbf{p}_i))$, where $F \in \mathcal{F}$ and $G \in \mathcal{G}$ [S3].

### SE3
SE3 is a method that addresses loop closure correction for implicit maps by implementing an equivariant mapping from SE3 to feature space, synchronously adjusting camera poses and implicit map representations [S4]. It leverages the frontend localization module of ORB-SLAM2 and uses a voxel grid as the backend to construct a SLAM system that achieves loop closure correction for implicit maps [S4].

## Background
Neural Implicit SLAM emerged from the integration of Neural Radiance Fields (NeRF) [S1] with traditional SLAM pipelines. Early works focused on offline reconstruction, but real-time RGB-D SLAM variants like the one described in [S1] introduced hybrid encodings (hash grids/tri-planes) to accelerate training. The problem formulation as a continuous online learning task with cached rays was formalized in [S3]. Variants like SLAIM [S1] and SNI-SLAM [S1] addressed specific issues like high-frequency rendering artifacts and semantic consistency. VI-NeRF-SLAM [S2] extended this to monocular/IMU scenarios with dynamic error correction. SE3 [S4] tackled the irreversibility of implicit maps for loop closure. The family is widely used in robotics for dense 3D mapping where explicit geometry is difficult to maintain [S1, S3].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Neural Radiance Field-based Visual Rendering: a Comprehensive Review, https://ieeexplore.ieee.org/abstract/document/11455380, fetched 2026-10-30
[S3] Benchmarking Implicit Neural Representation and Geometric Rendering in Real-Time RGB-D SLAM, https://doi.org/10.1109/cvpr52733.2024.02017, fetched 2026-10-26
[S4] Advancement in Neural Radiance Fields Propulsion for SLAM: A Novel Upgrade, https://www.atlantis-press.com/proceedings/icaic-24/126003439, fetched 2026-10-28
