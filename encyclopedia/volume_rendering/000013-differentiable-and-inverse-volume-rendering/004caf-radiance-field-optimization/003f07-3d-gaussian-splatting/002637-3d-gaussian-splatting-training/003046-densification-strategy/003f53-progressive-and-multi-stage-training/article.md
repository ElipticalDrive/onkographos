# Progressive and Multi-stage Training

Also called: Coarse-to-fine training, Curriculum learning for Gaussians, Two-stage optimization

## Overview
Progressive and multi-stage training methods structure the optimization of 3D Gaussian primitives into distinct phases to manage density, stability, and convergence [S1, unofficial][S2]. Unlike single-stage training, which often suffers from instability or suboptimal convergence in complex scenes, these approaches initialize a sparse or coarse representation and iteratively refine it through densification steps or alternating supervision signals [S1, unofficial][S2]. This family sets itself apart from sibling methods like [Score-based Densification](4705-score-based-densification.md) by focusing on the temporal evolution of the training process (phases, stages, or curricula) rather than solely on the spatial metrics used to select primitives for modification.

## Shared principles
The core mechanism of progressive and multi-stage training involves decoupling the optimization process into sequential or alternating phases, each with specific objectives or constraints [S1, unofficial][S2]. Common principles include:

1.  **Phased Initialization:** The training begins with a simplified or sparse representation (e.g., random points, coarse mesh, or feed-forward predictions) to establish a stable baseline before introducing complexity [S1, unofficial][S2].
2.  **Iterative Refinement:** Density or fidelity is increased gradually. This may involve splitting primitives, adding new ones, or switching supervision signals (e.g., from slices to voxels) as the model converges [S1, unofficial][S2].
3.  **Stability Management:** By separating initialization from fine-grained optimization, these methods prevent the optimizer from getting stuck in poor local minima caused by high-frequency noise or insufficient initial coverage [S1, unofficial][S2].
4.  **Hybrid Supervision:** Some variants alternate between different types of loss functions or data representations (e.g., 2D slices vs. 3D voxels) to balance local detail and global consistency [S1, unofficial].

## Subtopics

### [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)
A methodology that divides optimization into distinct phases to manage primitive density and stability. It initializes a coarse representation (sparse points or mesh) and progressively refines it through densification steps. It sets itself apart from siblings like [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md) by focusing on the structural evolution of the primitive population rather than alternating supervision signals.

### [Training Objective and Densification](7448-training-objective-and-densification.md)
Defines the optimization objective and densification strategy for **MeshGS**, a variant that aligns primitives with an explicit mesh surface. It solves the problem of Gaussian splats drifting away from geometry by introducing projection and scale regularization for "tightly-bound" splats, while maintaining standard densification for "loosely-bound" splats. It differs from standard 3DGS by explicitly separating splats into two populations based on their distance to the mesh.

### [Fine-tuning](9138-fine-tuning.md)
A training phase that refines an initial set of Gaussian primitives, typically generated via a feed-forward process, to achieve photorealistic novel view synthesis. It distinguishes itself from standard training by integrating densification and direct attribute optimization into a unified stage that leverages pretrained priors for faster convergence and reduced memory consumption. Unlike methods that rely solely on static densification heuristics, fine-tuning actively manages the primitive population to eliminate redundancy while preserving high-fidelity details.

### [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md)
Structures the training of 3D Gaussian representations for volumetric data by alternating between slice-based and voxel-based supervision signals. It solves the instability and incomplete coverage issues inherent in using either signal in isolation: slice supervision alone lacks global spatial guidance, while random voxel sampling alone leads to unstable global approximation. It sets itself apart from siblings like [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md) by explicitly leveraging the structural alignment between slice sampling and shear-warp rendering.

### Progressive Propagation Strategy
Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage [S1, unofficial]. It computes a local perceptual error map using a lightweight vision-transformer head and densifies only in regions where SSIM falls below a dynamic threshold, significantly improving detail in visually salient areas [S1, unofficial].

### Interleaved Densification
Alternates between spatial and temporal densification every 2 epochs to avoid Gaussians failing to reconstruct either motion or appearance detail [S2]. This is particularly useful in scenes with oscillatory motion where static and dynamic elements must be balanced [S2].

### Two-Stage Training Scheme for FWD-U
A training strategy that decouples depth network initialization from full model training to improve stability and performance [S3].
*   **Stage 1:** Construct RGB point clouds for each input view. Project the constructed point clouds to the target view using the differentiable point cloud renderer. Aggregate all point clouds from the various input views to render RGB images at the target view. Optimize the depth network using a photometric loss between the rendered target views and the ground truth target images.
*   **Stage 2:** Freeze or continue training the initialized depth network and jointly train the entire model (spatial feature network, MLP, renderer, transformer, refinement module).
*   **Parameter:** Performance improvement of approximately 0.4 dB PSNR gain [S3].
*   **Pitfall:** Jointly training the depth network and other components from scratch causes instability [S3].

## Background
The concept of progressive training in radiance fields originates from the need to stabilize the optimization of high-dimensional, non-convex loss landscapes [S1, unofficial]. Early 3D Gaussian Splatting implementations used a single-stage optimization with fixed densification schedules [S1, unofficial]. However, researchers observed that initializing with a sparse set of primitives or using curriculum learning (e.g., starting with low-resolution or simplified supervision) led to better convergence and higher final fidelity [S1, unofficial][S2]. The Hybrid Coarse-to-Fine Strategy and MeshGS variants emerged to address specific geometric constraints, while curriculum learning for volumetric data addressed the unique challenges of 3D medical or scientific imaging [S1, unofficial][S2]. The Two-Stage Training Scheme for FWD-U highlights the broader applicability of staged optimization to depth estimation in multi-view stereo tasks [S3].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
[S3] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
