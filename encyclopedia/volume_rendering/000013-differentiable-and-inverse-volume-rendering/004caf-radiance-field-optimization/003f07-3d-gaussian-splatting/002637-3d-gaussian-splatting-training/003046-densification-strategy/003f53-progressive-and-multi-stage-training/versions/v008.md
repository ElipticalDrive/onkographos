# Progressive and Multi-stage Training

Also called: Coarse-to-fine training, Curriculum learning for 3DGS, Staged optimization

## Overview
Progressive and multi-stage training addresses the instability and local minima traps inherent in optimizing 3D Gaussian Splatting (3DGS) by partitioning the optimization process into sequential phases of refinement, rather than training the point cloud in a single static phase [S1, unofficial]. It solves the problem of poor geometric initialization and insufficient detail recovery by progressively densifying the representation based on error metrics or switching supervision signals [S1, unofficial]. It yields significantly better rendering quality than single-stage training by decoupling global structure learning from local detail recovery [S1, unofficial].

## Shared principles
The family of progressive and multi-stage training methods shares the following mechanism:

1.  **Phase Decomposition:** The total training budget (iterations or epochs) is divided into distinct stages. Each stage has a specific objective, such as establishing global geometry, refining local texture, or managing primitive density [S1, unofficial].
2.  **Progressive Densification:** Instead of initializing with a dense cloud, methods often start with a sparse representation (e.g., SfM points or mesh vertices) and iteratively increase density. This is triggered by specific metrics (e.g., DSSIM error) or fixed iteration intervals [S1, unofficial].
3.  **Curriculum or Hierarchical Supervision:** Some methods alternate between different types of supervision signals (e.g., sparse voxel supervision for global consistency vs. slice-based supervision for local detail) to stabilize learning [S1, unofficial].
4.  **Attribute Refinement:** Later stages typically focus on optimizing high-frequency attributes (color, fine-scale geometry) while keeping the primitive count relatively stable or pruning redundancies [S1, unofficial].

## Subtopics

### [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)
A training strategy that partitions optimization into sequential stages of refinement, starting from a sparse mesh or point cloud and densifying based on structural dissimilarity (DSSIM) [7173-hybrid-coarse-to-fine-strategy.md]. It sets itself apart by using a mesh-based variant where triangles are split based on an area-weighted error score, preventing repeated splitting of tiny triangles [7173-hybrid-coarse-to-fine-strategy.md]. Choose this when working with explicit mesh representations or when edge-aware initialization is required [7173-hybrid-coarse-to-fine-strategy.md].

### [Training Objective and Densification](7448-training-objective-and-densification.md)
Defines the loss function and densification strategy for MeshGS, a variant that aligns primitives with an explicit mesh surface [7448-training-objective-and-densification.md]. It solves the problem of Gaussian splats drifting away from geometry by introducing projection and normal consistency losses for "tightly-bound" splats, while loosely-bound splats follow standard 3DGS rules [7448-training-objective-and-densification.md]. Choose this when integrating 3DGS with explicit mesh geometry [7448-training-objective-and-densification.md].

### [Fine-tuning](9138-fine-tuning.md)
The final optimization stage in 3DGS pipelines, designed to refine geometry and color attributes of a pre-initialized point cloud to achieve photorealistic novel view synthesis [9138-fine-tuning.md]. It builds on [Progressive and Multi-stage Training](9966-progressive-and-multi-stage-training.md) by applying standard differentiable rendering optimization to a densified cloud [9138-fine-tuning.md]. Unlike coarse stages, it targets high-frequency detail recovery and color accuracy, often leveraging pretrained priors [9138-fine-tuning.md]. Choose this as the final step after densification is complete [9138-fine-tuning.md].

### [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md)
Addresses instability in volumetric 3DGS by decoupling global volumetric consistency from local structural detail learning [12921-curriculum-learning-with-slice-and-voxel-supervision.md]. It sets itself apart by explicitly alternating between sparse voxel supervision for global geometry and structured slice supervision for local texture, rather than relying solely on pixel-level photometric loss [12921-curriculum-learning-with-slice-and-voxel-supervision.md]. Choose this for volumetric data (e.g., MRI, Cryosection) where global spatial guidance is critical [12921-curriculum-learning-with-slice-and-voxel-supervision.md].

### Progressive Propagation Strategy
Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage [S1, unofficial]. It expands the cloud in a fronto-parallel manner to improve reconstruction quality in visually salient areas [S1, unofficial].

### Interleaved Densification
Alternates between spatial and temporal densification every 2 epochs to avoid Gaussians failing to reconstruct either motion or appearance detail in dynamic scenes [S2]. This prevents the optimization landscape from becoming unstable due to conflicting spatial and temporal gradients [S2].

## Background
The concept of progressive training in 3DGS originates from the observation that initializing with a dense cloud can lead to local minima and inefficient optimization [S1, unofficial]. The original 3DGS paper [S1, unofficial] introduced a densification strategy that starts with SfM points and adds Gaussians based on gradient norms. Subsequent works like GaussianPro [S1, unofficial] proposed progressive propagation for smoother expansion. MeshGS [7448-training-objective-and-densification.md] adapted this for mesh-aligned splats. Curriculum learning approaches [12921-curriculum-learning-with-slice-and-voxel-supervision.md] were developed for volumetric data to handle the complexity of 3D structures. Dynamic scene methods like Adaptive Spatio-Temporal 3DGS [S2] introduced interleaved densification to handle motion.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
