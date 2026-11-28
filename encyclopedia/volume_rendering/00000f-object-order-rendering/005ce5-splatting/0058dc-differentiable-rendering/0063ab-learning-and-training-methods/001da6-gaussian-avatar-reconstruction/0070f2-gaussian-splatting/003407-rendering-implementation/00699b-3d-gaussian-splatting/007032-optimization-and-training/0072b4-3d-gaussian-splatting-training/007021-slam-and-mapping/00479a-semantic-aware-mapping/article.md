# Semantic-aware mapping
## Overview
Semantic-aware mapping integrates semantic information into the simultaneous localization and mapping (SLAM) process to guide the optimization of 3D scene representations, such as 3D Gaussians or Neural Radiance Fields (NeRFs) [S1, S2]. It solves the problem of geometric ambiguity and dynamic object interference by distilling 2D semantic labels to constrain 3D structure, enabling semantic-aware mapping [S2]. It distinguishes itself from sibling methods like [CG-SLAM](9196-cg-slam.md) and [Neural Implicit Representation](22462-neural-implicit-representation.md) by explicitly leveraging class-wise scene decomposition or semantic priors to distinguish static from dynamic elements or to guide Gaussian splitting, rather than relying solely on photometric or geometric constraints [S1, S2].
## Shared principles
The family shares the mechanism of integrating 2D semantic segmentation outputs into the 3D reconstruction pipeline. Common terms include "semantic distillation," "semantic priors," and "class-wise decomposition." The core mathematical principle involves a combined loss function:
$$
\mathcal{L} = \mathcal{L}_{rgb} + \lambda_{sem} \mathcal{L}_{sem}
$$
where $\mathcal{L}_{rgb}$ is the photometric reconstruction loss, $\lambda_{sem}$ is the semantic loss weight, and $\mathcal{L}_{sem}$ measures the discrepancy between rendered semantic features (or projected labels) and 2D semantic masks [S2]. This allows gradients from semantic consistency to update Gaussian parameters (position, scale, rotation, opacity, color, semantics) and camera poses [S2].
## Subtopics

### [SemGauss-SLAM](9207-semgauss-slam.md)
[SemGauss-SLAM](9207-semgauss-slam.md) is a SLAM method that integrates semantic information into the 3D Gaussian Splatting framework. It distinguishes itself from standard radiance-field optimization by distilling 2D semantic labels to guide the Gaussian representation, enabling semantic-aware mapping. It is chosen when semantic consistency is required to prevent redundant splitting in homogeneous regions and to refine camera poses using semantic cues.

### [Unsupervised Asset Grouping](20077-unsupervised-asset-grouping.md)
[Unsupervised Asset Grouping](20077-unsupervised-asset-grouping.md) is a method for categorizing discovered 3D objects by comparing the distributions of color and semantic features among patches using Jensen-Shannon Divergence (JSD) and agglomerative clustering. It solves the problem of object categorization in the absence of explicit object pose information, where pixel-wise comparison is suboptimal. It sets itself apart from sibling methods like SemGauss-SLAM by focusing on post-optimization asset mining and reconfiguration via feature plane discretization rather than joint semantic-geometric optimization during rendering.

### NEDS-SLAM
- [S1] A SLAM method that considers Gaussian’s semantic information in the simultaneous localization and mapping process by distilling 2D semantic information which can be obtained using 2D segmentation methods or provided by the dataset.

### DDN-SLAM
- [S2] A SLAM system that integrates semantic priors with NeRF-based representation to distinguish between dynamic and static objects.
- [S2] Separates tracking and NeRF-based mapping into four threads: segmentation, tracking, mapping, and loop detection.
- [S2] Segmentation thread identifies and suppresses dynamic feature points and regions.
- [S2] Tracking thread extracts features, filters them via semantic and geometric cues, computes static optical flow, and produces camera poses and keyframes.
- [S2] Mapping thread integrates input sparse point cloud to guide NeRF-based ray sampling and uses dynamic-aware masks to drive keyframe selection and volume rendering, preserving static surface geometry.
- [S2] Loop detection thread detects revisited areas and performs global bundle adjustment, enhancing long-range consistency.

### DNS-SLAM
- [S2] A dense semantic-aware SLAM framework using class-wise scene decomposition with a hybrid point-based Neural Field mapping module.
- [S2] Introduces a multi-class neural scene representation that explicitly links object classes to NeRF camera poses.
- [S2] Leverages 2D semantic priors and multiview image features to strengthen pose estimation through back-projected geometric constraints.
- [S2] Trains a lightweight coarse model via self-supervision to accelerate tracking.
- [S2] Refines geometry by supervising occupancy with Gaussian-distributed priors.

### vMAP
- [S3] A vectorized object-level mapping where each object is detected through its 3D point cloud and instance segmentation map, represented by a separate MLP.
- [S3] Continually update 3D bounds via data association across frames.
- [S3] Leads to improvements in object-level reconstruction quality and runtime efficiency compared to traditional SLAM systems.

## Background
Semantic-aware mapping in the context of Gaussian Splatting and NeRF-based SLAM emerged as a response to the limitations of purely geometric or photometric optimization, particularly in dynamic scenes or textureless regions. Early works like LIVGaussMap [184] initialized Gaussians with LiDAR, while subsequent methods such as SGS-SLAM [185], NEDS-SLAM [186], and SemGauss-SLAM [187] integrated 2D semantic information to improve mapping consistency [S1]. The field expanded to include NeRF-based approaches like DDN-SLAM [251] and DNS-SLAM [254], which use semantic priors to handle dynamic objects and class-wise decomposition [S2]. Object-level mapping methods like vMAP [105] further refined this by representing objects as separate MLPs with continually updated 3D bounds [S3]. These methods are used in robotics for relocalization, navigation, and 6D pose estimation [S1].
## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S3] Semantically-aware Neural Radiance Fields for Visual Scene Understanding: A Comprehensive Review, https://link.springer.com/article/10.1007/s11263-025-02663-5, fetched 2026-10-27
