# Progressive and Multi-stage Training

Also called: Coarse-to-fine training, Stage-wise optimization, Curriculum learning for 3DGS

## Overview
Progressive and Multi-stage Training refers to methodologies that decouple the optimization of 3D Gaussian Splatting (3DGS) into sequential phases or stages to stabilize convergence, improve rendering quality, and manage primitive density [S1, unofficial]. Unlike single-stage training, which optimizes the point cloud from initialization to final density in one continuous process, these methods employ distinct phases such as coarse initialization followed by fine-tuning, or hierarchical supervision signals [S1, unofficial]. This approach sets itself apart from sibling methods like [Score-based Densification](4705-score-based-densification.md) by focusing on the structural refinement of the point cloud's spatial distribution and density through staged optimization rather than solely on per-step error metrics.

## Shared principles
The core mechanism involves dividing the training process into distinct temporal or structural stages, where the objective, initialization, or supervision signal changes between stages. Common principles include:

1.  **Stage Decoupling:** The training loop is partitioned into phases (e.g., Coarse, Fine, or Stage 1, Stage 2). Parameters or network components may be frozen, initialized differently, or optimized with different loss weights in each stage [S1, unofficial][S3].
2.  **Progressive Densification:** Instead of random or uniform densification, new primitives are introduced based on error metrics or structural requirements at specific stages. For example, a coarse representation is established first, and densification occurs only after the global structure converges [S1, unofficial][S2].
3.  **Hierarchical Supervision:** In volumetric or complex scenes, supervision signals are applied in a specific order (e.g., local slice constraints before global voxel constraints) to prevent instability [S1, unofficial].
4.  **Stabilization via Initialization:** Multi-stage schemes often use a simpler sub-problem (e.g., depth network initialization via photometric loss) to provide a stable starting point for the full model, avoiding instability from joint training from scratch [S3].

## Subtopics

### [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)
Decouples optimization into a coarse initialization phase (using sparse sampling and Delaunay triangulation for mesh variants) and a fine-tuning phase with error-based densification.

### [Training Objective and Densification](7448-training-objective-and-densification.md)
Defines mesh-aware losses and a dual-strategy densification process for MeshGS, treating tightly-bound and loosely-bound splats differently.

### [Fine-tuning](9138-fine-tuning.md)
An optimization phase following initialization (often feed-forward) that jointly optimizes all attributes while actively managing primitive count through growing and pruning.

### [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md)
Stabilizes volumetric optimization by applying planar slice constraints first, followed by sparse voxel constraints for global consistency.

### Progressive Propagation Strategy
- **Definition:** Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage [S1, unofficial].
- **Mechanism:** Expands the Gaussian cloud in a fronto-parallel manner, likely guided by local perceptual error maps to densify only where SSIM falls below a dynamic threshold [S1, unofficial].

### Interleaved Densification
- **Step:** Alternates between spatial and temporal densification every 2 epochs to avoid Gaussians failing to reconstruct either motion or appearance detail [S2].

### Two-Stage Training Scheme for FWD-U
- **Definition:** A training strategy that decouples depth network initialization from full model training to improve stability and performance [S3].
- **Step:**
    1.  Construct RGB point clouds for each input view.
    2.  Project the constructed point clouds to the target view using the differentiable point cloud renderer.
    3.  Aggregate all point clouds from the various input views to render RGB images at the target view.
    4.  Optimize the depth network using a photometric loss between the rendered target views and the ground truth target images.
    5.  Freeze or continue training the initialized depth network and jointly train the entire model (spatial feature network, MLP, renderer, transformer, refinement module).
- **Parameter:** Performance improvement: approximately 0.4 dB PSNR gain [S3].
- **Pitfall:** Jointly training the depth network and other components from scratch causes instability [S3].

## Background
The concept of multi-stage training in 3D Gaussian Splatting emerged as a response to the instability and inefficiency of optimizing dense point clouds from sparse or random initializations. Early works noted that joint training of complex components (like depth networks in forward-warping methods) from scratch leads to instability, necessitating initialization stages [S3]. Similarly, in mesh-based and volumetric variants, the need to balance local detail reconstruction with global structural consistency led to curriculum learning approaches [S1, unofficial][S2]. The "Coarse-to-Fine" paradigm is widely adopted in computer vision for radiance fields and point clouds to ensure that global geometry is established before fine details are added [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
[S3] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
