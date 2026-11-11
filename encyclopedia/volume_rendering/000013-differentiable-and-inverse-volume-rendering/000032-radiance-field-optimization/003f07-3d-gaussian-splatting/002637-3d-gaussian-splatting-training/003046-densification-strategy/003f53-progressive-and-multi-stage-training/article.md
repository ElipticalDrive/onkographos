# Progressive and Multi-stage Training

Also called: Coarse-to-fine training, Curriculum learning for Gaussians, Progressive densification

## Overview
Progressive and multi-stage training strategies address the instability and local minima convergence issues inherent in optimizing 3D Gaussian Splatting (3DGS) from scratch by structuring the optimization process into distinct phases or stages of increasing complexity [S1, unofficial]. Unlike single-stage training, which optimizes a static or uniformly densified point cloud, these methods initialize with a coarse representation (sparse points, feed-forward predictions, or mesh-aligned primitives) and iteratively refine density and detail based on reconstruction error or geometric priors [7173-hybrid-coarse-to-fine-strategy.md][9138-fine-tuning.md]. This approach yields significantly higher rendering quality and faster convergence by ensuring the Gaussian cloud expands smoothly and covers salient regions effectively [S1, unofficial][9138-fine-tuning.md].

## Shared principles
The core mechanism of progressive and multi-stage training in 3DGS involves decoupling the initialization of the Gaussian cloud from the fine-grained optimization, often introducing intermediate stages that enforce specific structural or density constraints before allowing full freedom [7173-hybrid-coarse-to-fine-strategy.md].

1.  **Staged Initialization:** The training does not begin with the final target density. Instead, it starts with a sparse set of primitives (e.g., SfM points, mesh vertices, or feed-forward predictions) that provide a rough geometric and color approximation [7173-hybrid-coarse-to-fine-strategy.md][9138-fine-tuning.md].
2.  **Error-Driven Densification:** New primitives are added not randomly, but in regions where the current representation fails to reconstruct the input data. Metrics such as DSSIM, gradient magnitude, or reconstruction error trigger the insertion of new Gaussians [7173-hybrid-coarse-to-fine-strategy.md][9138-fine-tuning.md].
3.  **Curriculum or Phase-Based Supervision:** Some methods enforce a specific order of supervision signals. For example, local structural constraints (like slice-based losses) are applied before global volumetric consistency (voxel-based losses) to stabilize optimization [12921-curriculum-learning-with-slice-and-voxel-supervision.md].
4.  **Pruning for Efficiency:** As the cloud densifies, redundant or low-opacity primitives are removed to maintain computational efficiency and prevent overfitting [7173-hybrid-coarse-to-fine-strategy.md][9138-fine-tuning.md].

## Subtopics

### [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)
Refines primitive distribution through multiple stages of increasing density, starting with a sparse structure and iteratively adding primitives based on error metrics. It addresses convergence to local minima with insufficient geometric detail. For mesh-based variants, it uses DSSIM and area-weighting to score triangles for splitting.

### [Training Objective and Densification](7448-training-objective-and-densification.md)
Defines the loss function and densification strategy for **MeshGS**, a variant aligning primitives with an explicit mesh surface. It solves Gaussian drift by splitting primitives into tightly-bound and loosely-bound sets, applying mesh-specific regularization (normal consistency, projection) to the former and using mesh-based sampling for densification.

### [Fine-tuning](9138-fine-tuning.md)
Initializes the scene using a direct feed-forward prediction of geometry and color attributes, followed by differentiable optimization. It solves slow convergence and high memory overhead by leveraging pretrained priors to generate a redundant but high-quality initial set, which is then optimized and pruned. It differs from coarse-to-fine by replacing manual initialization with a learned prior.

### [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md)
Addresses instability in shear-warp rendering by enforcing a structured training schedule. It sets itself apart by leveraging geometric alignment between data sampling and the rendering pipeline, using planar slice constraints to guide local structure before establishing global volumetric consistency via voxel supervision.

### Progressive Propagation Strategy
- **Definition:** Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage [S1, unofficial]. It expands the cloud in a fronto-parallel manner, preventing abrupt density changes that can destabilize optimization.

### Interleaved Densification
- **Step:** Alternates between spatial and temporal densification every 2 epochs to avoid Gaussians failing to reconstruct either motion or appearance detail in dynamic scenes [S2]. This prevents the optimizer from over-committing to static geometry at the expense of temporal coherence or vice versa.

## Background
The concept of coarse-to-fine optimization is well-established in computer graphics and neural rendering. In the context of 3D Gaussian Splatting, the original method [22] uses a fixed densification schedule based on gradient norms. However, this can lead to artifacts or incomplete reconstructions in complex scenes. Subsequent works like **MeshGS** [7448-training-objective-and-densification.md] and **EVolSplat** [9138-fine-tuning.md] introduced structured initialization and multi-stage objectives to improve robustness. The **Hybrid Coarse-to-Fine Strategy** [7173-hybrid-coarse-to-fine-strategy.md] explicitly formalized the iterative error-driven densification for both point clouds and meshes. **GaussianPro** [S1, unofficial] proposed progressive propagation for smoother expansion. For dynamic scenes, **Adaptive Spatio-Temporal 3DGS** [S2] introduced interleaved densification to handle oscillatory motion. These methods collectively shift the paradigm from "optimize everything everywhere" to "start coarse, refine locally, prune globally."

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
