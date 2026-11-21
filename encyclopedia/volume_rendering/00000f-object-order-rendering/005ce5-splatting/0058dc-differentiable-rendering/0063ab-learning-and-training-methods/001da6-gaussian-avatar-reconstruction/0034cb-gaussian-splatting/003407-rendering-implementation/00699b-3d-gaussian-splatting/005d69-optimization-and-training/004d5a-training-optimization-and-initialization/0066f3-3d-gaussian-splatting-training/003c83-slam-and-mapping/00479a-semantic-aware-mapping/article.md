# Semantic-aware mapping

Also called: Semantic 3D Gaussian Splatting SLAM, Semantic-aware Gaussian Mapping

## Overview
Semantic-aware mapping extends 3D Gaussian Splatting (3DGS) SLAM by integrating 2D semantic segmentation priors into the optimization loop to produce semantically labeled 3D maps [S1]. It distinguishes itself from standard radiance-field optimization by distilling 2D semantic labels to guide the Gaussian representation, enabling semantic-aware mapping [S2]. This family addresses the limitation of purely geometric SLAM by providing class-level understanding of the scene, which aids in dynamic object handling, loop closure, and high-level reasoning.

## Shared principles
The family builds on [SLAM and Mapping](9268-slam-and-mapping.md). The core mechanism involves augmenting the standard 3D Gaussian parameter set with semantic attributes and modifying the loss function to include semantic consistency terms.

1.  **Semantic Augmentation:** Each 3D Gaussian $i$ is assigned a semantic label $l_i$. This label can be a discrete class identifier or a probability vector over semantic classes [S2].
2.  **Semantic Distillation:** 2D semantic masks are obtained from a pre-trained 2D segmentation network or dataset annotations [S2]. These 2D labels are projected or back-projected to align with the 3D Gaussians.
3.  **Semantic Loss:** A semantic distillation loss $\mathcal{L}_{sem}$ is computed between the rendered semantic features (or projected labels) and the 2D semantic masks [S2].
4.  **Combined Optimization:** The total loss $\mathcal{L}$ combines photometric reconstruction loss $\mathcal{L}_{rgb}$ and the semantic loss:
    $$
    \mathcal{L} = \mathcal{L}_{rgb} + \lambda_{sem} \mathcal{L}_{sem}
    $$
    where $\lambda_{sem}$ is the weight for the semantic loss [S2]. Gradients from both terms are backpropagated to update Gaussian parameters (position, scale, rotation, opacity, color, semantics) and camera poses [S2].
5.  **Semantic-Guided Control:** Adaptive control mechanisms (splitting and pruning) may be guided by semantic boundaries to avoid redundant splitting in homogeneous semantic regions [S1].

## Subtopics

### [SemGauss-SLAM](9207-semgauss-slam.md)
[SemGauss-SLAM](9207-semgauss-slam.md) is a SLAM method that integrates semantic information into the 3D Gaussian Splatting framework. It distinguishes itself from standard radiance-field optimization by distilling 2D semantic labels to guide the Gaussian representation, enabling semantic-aware mapping. It is chosen when a dense, semantically labeled 3D map is required for tasks like navigation or scene understanding.

### [Unsupervised Asset Grouping](20077-unsupervised-asset-grouping.md)
[Unsupervised Asset Grouping](20077-unsupervised-asset-grouping.md) is a method for categorizing discovered objects in a scene by comparing the distributions of color and semantic features among image patches using Jensen-Shannon Divergence (JSD) and agglomerative clustering. It sets itself apart from sibling methods like [SemGauss-SLAM](9207-semgauss-slam.md) by operating on a discretized ground feature plane to group assets without requiring explicit object pose information or supervised labels, enabling the construction of a cross-scene asset library.

### NEDS-SLAM
#### definition
- [S1] A SLAM method that considers Gaussian’s semantic information in the simultaneous localization and mapping process by distilling 2D semantic information which can be obtained using 2D segmentation methods or provided by the dataset.

### DDN-SLAM
#### definition
- [S2] A SLAM system that integrates semantic priors with NeRF-based representation to distinguish between dynamic and static objects.
#### step
- [S2] Separates tracking and NeRF-based mapping into four threads: segmentation, tracking, mapping, and loop detection.
- [S2] Segmentation thread identifies and suppresses dynamic feature points and regions.
- [S2] Tracking thread extracts features, filters them via semantic and geometric cues, computes static optical flow, and produces camera poses and keyframes.
- [S2] Mapping thread integrates input sparse point cloud to guide NeRF-based ray sampling and uses dynamic-aware masks to drive keyframe selection and volume rendering, preserving static surface geometry.
- [S2] Loop detection thread detects revisited areas and performs global bundle adjustment, enhancing long-range consistency.

### DNS-SLAM
#### definition
- [S2] A dense semantic-aware SLAM framework using class-wise scene decomposition with a hybrid point-based Neural Field mapping module.
#### step
- [S2] Introduces a multi-class neural scene representation that explicitly links object classes to NeRF camera poses.
- [S2] Leverages 2D semantic priors and multiview image features to strengthen pose estimation through back-projected geometric constraints.
- [S2] Trains a lightweight coarse model via self-supervision to accelerate tracking.
- [S2] Refines geometry by supervising occupancy with Gaussian-distributed priors.

## Background
The integration of semantic information into 3D Gaussian Splatting for SLAM emerged as a natural extension of both semantic SLAM and 3DGS. Early works like LIVGaussMap [184] focused on geometric initialization using LiDAR. Subsequently, methods such as SGS-SLAM [185], NEDS-SLAM [186], and SemGauss-SLAM [187] incorporated semantic information by distilling 2D semantic labels [S1]. Concurrently, other approaches like DDN-SLAM [251] and DNS-SLAM [254] explored semantic priors within NeRF-based or hybrid neural field representations to handle dynamic scenes and improve pose estimation [S2]. These methods aim to enhance the robustness of SLAM in complex environments by leveraging high-level semantic cues alongside low-level photometric and geometric constraints.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
