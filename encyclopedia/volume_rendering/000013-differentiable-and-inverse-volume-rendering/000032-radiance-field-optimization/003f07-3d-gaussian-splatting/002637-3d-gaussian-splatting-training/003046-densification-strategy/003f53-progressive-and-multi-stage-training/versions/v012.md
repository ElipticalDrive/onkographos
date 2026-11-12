# Progressive and Multi-stage Training

Also called: Coarse-to-Fine Training, Curriculum Learning for Gaussians, Two-Stage Optimization

## Overview
Progressive and Multi-stage Training refers to methodologies that divide the optimization of 3D Gaussian representations into distinct phases or stages, rather than training the primitive set in a single uniform pass [S1, unofficial]. It solves the instability, poor convergence, and structural incompleteness inherent in single-stage optimization by first establishing a global structural approximation or coarse geometry, and then progressively refining local details through densification or fine-tuning [S1, unofficial]. This approach yields significantly higher rendering quality compared to single-stage baselines, as evidenced by improved PSNR, SSIM, and LPIPS metrics on standard benchmarks [S1, unofficial].

## Shared principles
The family shares the principle of decoupling the optimization process into temporally or structurally distinct phases to manage the complexity of the scene representation.

1.  **Phase Separation:** The training process is divided into at least two stages. The initial stage focuses on global structure, coarse geometry, or initialization, while subsequent stages focus on high-frequency detail, fine-tuning, or densification [S1, unofficial].
2.  **Progressive Densification:** In coarse-to-fine strategies, the density of primitives is increased iteratively. The initial state is sparse, and densification triggers (splitting/cloning) are applied only after a warm-up period or specific iteration thresholds are met [S1, 7173-hybrid-coarse-to-fine-strategy.md, unofficial].
3.  **Curriculum Supervision:** Some methods employ a curriculum where the supervision signal changes over time. For example, supervision may start with planar (slice) constraints to establish continuity before introducing volumetric (voxel) constraints for global guidance [12921-curriculum-learning-with-slice-and-voxel-supervision.md].
4.  **Stabilization via Initialization:** Multi-stage schemes often use the first stage to provide a stable initialization for difficult components (e.g., depth networks or feed-forward geometry predictions), preventing the instability associated with joint training from scratch [S3, 9138-fine-tuning.md].

## Subtopics

### [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)
*   **What it is:** A methodology that divides optimization into coarse and fine stages, initializing a sparse representation and iteratively densifying it based on error metrics [7173-hybrid-coarse-to-fine-strategy.md].
*   **Differentiation:** Unlike standard single-stage training, it explicitly separates the establishment of global structure from local refinement. It differs from [Adaptive Density Control](9188-adaptive-density-control.md) by using a specific triangle-based scoring metric (DSSIM weighted by area) for mesh/point-cloud based initialization, whereas 3DGS uses gradient norms on Gaussians [7173-hybrid-coarse-to-fine-strategy.md].
*   **When to choose:** When starting from a sparse mesh or point cloud, or when single-stage training fails to converge to a high-quality global structure [7173-hybrid-coarse-to-fine-strategy.md].
*   **Link:** [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)

### [Training Objective and Densification](7448-training-objective-and-densification.md)
*   **What it is:** A method for **MeshGS** that integrates explicit mesh alignment into the training loop, classifying Gaussians as "tightly-bound" or "loosely-bound" relative to a mesh surface [7448-training-objective-and-densification.md].
*   **Differentiation:** Unlike [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md) which focuses on resolution scaling, this method focuses on geometric fidelity through mesh alignment. It introduces geometry-aware losses (normal consistency, projection, scale regularization) and a dual densification strategy [7448-training-objective-and-densification.md].
*   **When to choose:** When the scene has known explicit geometry (mesh) and Gaussian primitives must be constrained to adhere to it, preventing drift or uncontrolled expansion [7448-training-objective-and-densification.md].
*   **Link:** [Training Objective and Densification](7448-training-objective-and-densification.md)

### [Fine-tuning](9138-fine-tuning.md)
*   **What it is:** The phase where a pre-initialized set of Gaussian primitives (e.g., from a feed-forward network) is optimized to achieve high-fidelity novel view synthesis [9138-fine-tuning.md].
*   **Differentiation:** It sets itself apart from standard training by explicitly enabling growing and pruning during the optimization process to discard redundant initial Gaussians while refining attributes. It leverages pretrained priors for faster convergence and reduced memory consumption compared to training from scratch [9138-fine-tuning.md].
*   **When to choose:** When using a feed-forward initialization or when memory constraints require pruning redundant primitives from a coarse initialization [9138-fine-tuning.md].
*   **Link:** [Fine-tuning](9138-fine-tuning.md)

### [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md)
*   **What it is:** A method for volumetric data (e.g., MRI) that enforces a structured curriculum of supervision signals, alternating between planar (slice) and volumetric (voxel) constraints [12921-curriculum-learning-with-slice-and-voxel-supervision.md].
*   **Differentiation:** It sets itself apart from siblings like [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md) by explicitly alternating supervision types to align with shear-warp rendering, rather than relying solely on geometric density metrics. It solves the instability of random voxel sampling and the lack of global guidance in slice-only supervision [12921-curriculum-learning-with-slice-and-voxel-supervision.md].
*   **When to choose:** When optimizing 3D Gaussian representations for volumetric data where global spatial guidance is critical for structural completeness [12921-curriculum-learning-with-slice-and-voxel-supervision.md].
*   **Link:** [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md)

### Progressive Propagation Strategy
*   **Definition:** Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage [S1, unofficial].
*   **Details:** It expands the Gaussian cloud in a fronto-parallel manner [S1, unofficial].

### Interleaved Densification
*   **Step:** Alternate between spatial and temporal densification every 2 epochs to avoid Gaussians failing to reconstruct either motion or appearance detail [S2].

### Two-Stage Training Scheme for FWD-U
*   **Definition:** A training strategy that decouples depth network initialization from full model training to improve stability and performance [S3].
*   **Step:**
    1.  Stage 1: Construct RGB point clouds for each input view.
    2.  Stage 1: Project the constructed point clouds to the target view using the differentiable point cloud renderer.
    3.  Stage 1: Aggregate all point clouds from the various input views to render RGB images at the target view.
    4.  Stage 1: Optimize the depth network using a photometric loss between the rendered target views and the ground truth target images.
    5.  Stage 2: Freeze or continue training the initialized depth network and jointly train the entire model (spatial feature network, MLP, renderer, transformer, refinement module) [S3].
*   **Parameter:** Performance improvement: approximately 0.4 dB PSNR gain [S3].
*   **Pitfall:** Jointly training the depth network and other components from scratch causes instability [S3].

## Background
The concept of coarse-to-fine optimization is prevalent in computer graphics and neural rendering to handle the complexity of high-frequency details. In the context of 3D Gaussian Splatting, single-stage training can suffer from instability, particularly when initializing from sparse data or when dealing with complex geometries [7173-hybrid-coarse-to-fine-strategy.md]. The Hybrid Coarse-to-Fine Strategy, derived from earlier work on differentiable mesh rendering, adapts this concept to primitive-based reconstruction [7173-hybrid-coarse-to-fine-strategy.md]. Curriculum learning approaches for volumetric data address specific instabilities in 3D Gaussian optimization for medical or scientific volumes [12921-curriculum-learning-with-slice-and-voxel-supervision.md]. Multi-stage schemes for depth estimation and novel view synthesis, such as FWD-U, highlight the instability of joint training for depth networks, necessitating a two-stage approach [S3].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
[S3] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
