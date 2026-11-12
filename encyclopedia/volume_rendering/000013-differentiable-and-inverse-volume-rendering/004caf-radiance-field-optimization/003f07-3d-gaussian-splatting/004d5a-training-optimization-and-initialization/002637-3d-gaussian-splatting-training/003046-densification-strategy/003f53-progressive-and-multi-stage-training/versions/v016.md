# Progressive and Multi-stage Training

Also called: Multi-stage Optimization, Curriculum Learning for Gaussians, Coarse-to-Fine Training

## Overview
Progressive and Multi-stage Training refers to a family of optimization strategies that decouple the initialization, densification, and refinement of 3D Gaussian primitives into distinct phases or stages. It solves the problem of optimization instability and poor convergence that arises when attempting to simultaneously initialize a sparse representation, densify it to cover complex geometry, and optimize high-frequency attributes in a single monolithic process. Unlike single-stage training methods that optimize a static or dynamically densified set from scratch, this family sets itself apart by structuring the learning trajectory—either through coarse-to-fine geometric refinement, decoupled loss supervision, or staged network initialization—to achieve higher fidelity and stability.

## Shared principles
The core mechanism of progressive and multi-stage training involves structuring the optimization timeline $\mathcal{T}$ into distinct intervals or phases, each with specific objectives, constraints, or data sampling strategies.

1.  **Phase Decoupling:** The training process is divided into $N$ stages $S_1, S_2, \dots, S_N$. Each stage $S_i$ may have a different set of active parameters, loss functions, or densification rules. For example, early stages may focus on coarse geometry or specific loss terms, while later stages refine details or introduce additional constraints [S1, unofficial][S2, S3].
2.  **Progressive Densification/Refinement:** Instead of applying densification (splitting/cloning) uniformly or randomly, the density of primitives is increased progressively. This can be achieved by:
    *   Starting with a sparse initialization and densifying based on error metrics over time [S1, unofficial].
    *   Using a "fronto-parallel" expansion strategy to ensure smooth coverage [S1, unofficial].
    *   Alternating between spatial and temporal densification to handle dynamic scenes [S2].
3.  **Curriculum Supervision:** The supervision signal is introduced progressively. For instance, planar (slice) constraints may be applied before volumetric (voxel) constraints to establish local geometric continuity before global spatial guidance [S1, unofficial].
4.  **Parameter Freezing/Unfreezing:** In multi-stage schemes involving neural networks (e.g., depth estimation), earlier stages may train specific components (like a depth network) while freezing others, or use initialized weights for subsequent joint training to prevent instability [S3].

## Subtopics

### [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)
A multi-stage approach for point-based radiance fields that refines geometry and attributes iteratively. It addresses the difficulty of initializing and densifying point clouds simultaneously by decoupling coarse initialization from fine-grained refinement. It yields higher rendering quality, particularly in preserving high-frequency details.

### [Training Objective and Densification](7448-training-objective-and-densification.md)
Defines the loss and densification for **MeshGS**, a variant aligning primitives with an explicit mesh. It solves Gaussian drift by introducing surface-aware regularization. It differs from standard 3DGS by splitting primitives into tightly-bound ($G^t$) and loosely-bound ($G^l$) sets, applying mesh-aligned constraints only to the former.

### [Fine-tuning](9138-fine-tuning.md)
Refines a pre-initialized set of 3D Gaussians (e.g., from feed-forward predictions) to achieve high-fidelity synthesis. It sets itself apart by actively enabling growing and pruning during optimization, allowing the model to discard redundant initial Gaussians and adapt density to scene complexity, accelerating convergence compared to training from scratch.

### [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md)
Addresses instability in shear-warp rendering by decoupling supervision into planar (slice) and volumetric (voxel) constraints. It progressively introduces slice supervision to capture local geometric continuity before establishing global spatial guidance via voxel supervision, preventing unstable learning observed with random voxel sampling alone.

### Progressive Propagation Strategy
*   **Definition:** Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage [S1, unofficial]. It expands the Gaussian cloud in a fronto-parallel manner.

### Interleaved Densification
*   **Step:** Alternates between spatial and temporal densification every 2 epochs to avoid Gaussians failing to reconstruct either motion or appearance detail [S2].

### Two-Stage Training Scheme for FWD-U
*   **Definition:** A training strategy that decouples depth network initialization from full model training to improve stability and performance [S3].
*   **Step:**
    1.  Construct RGB point clouds for each input view.
    2.  Project the constructed point clouds to the target view using the differentiable point cloud renderer.
    3.  Aggregate all point clouds from the various input views to render RGB images at the target view.
    4.  Optimize the depth network using a photometric loss between the rendered target views and the ground truth target images.
    5.  Freeze or continue training the initialized depth network and jointly train the entire model (spatial feature network, MLP, renderer, transformer, refinement module).
*   **Parameter:** Performance improvement: approximately 0.4 dB PSNR gain [S3].
*   **Pitfall:** Jointly training the depth network and other components from scratch causes instability [S3].

## Background
The concept of multi-stage training in 3D Gaussian Splatting (3DGS) emerged as a response to the instability of optimizing dense Gaussian populations from sparse initializations. Standard 3DGS [Parent] uses a single-stage adaptive density control. However, variations like MeshGS [S1, unofficial] and FWD-U [S3] demonstrated that decoupling initialization from refinement or aligning with explicit geometry requires staged optimization. The Hybrid Coarse-to-Fine strategy [S1, unofficial] explicitly targets the optimization landscape difficulty of simultaneous densification. Curriculum learning approaches [S1, unofficial] were developed to handle the specific challenges of volumetric data (e.g., MRI) where global and local constraints interact non-trivially. Interleaved densification [S2] addresses the specific challenge of spatio-temporal consistency in dynamic scenes.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
[S3] FWD: Real-time Novel View Synthesis with Forward Warping and Depth, https://doi.org/10.1109/cvpr52688.2022.01526, fetched 2026-10-29
