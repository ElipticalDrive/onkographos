# Progressive and Multi-stage Training

Also called: Coarse-to-Fine Training, Curriculum Learning for Densification

## Overview
Progressive and Multi-stage Training improves the reconstruction quality of 3D Gaussian Splatting by dividing the optimization process into distinct phases, rather than training the point cloud in a single continuous pass [S1, unofficial]. It solves the problem of single-stage training failing to capture both global geometry and high-frequency details simultaneously by explicitly separating coarse initialization from fine-grained densification [S1, unofficial]. This approach yields significantly higher fidelity than single-stage training, particularly in preserving structural consistency and high-frequency details [S1, unofficial].

## Shared principles
The family relies on a staged optimization schedule where the density and complexity of the Gaussian representation evolve over time. Common principles include:
1.  **Staged Initialization:** The process begins with a sparse or coarse representation (e.g., SfM points, feed-forward predictions, or mesh vertices) that establishes global geometry and coarse color distribution [S1, unofficial].
2.  **Iterative Densification:** Densification is not performed uniformly. It is triggered periodically or conditionally after the coarse stage stabilizes, allowing the optimizer to converge on global structures before adding local detail [S1, unofficial].
3.  **Curriculum or Progressive Expansion:** Some variants use a curriculum (e.g., slice supervision before voxel supervision) or a progressive propagation strategy to ensure smooth expansion and better coverage, preventing instability caused by random sampling or lack of global guidance [S1, unofficial].
4.  **Fine-Tuning Phase:** A final stage refines the attributes (color, opacity, SH coefficients) of the densified primitives, often enabling growing and pruning to manage redundancy from initial estimates [S1, unofficial].

## Subtopics

### [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)
A training strategy that explicitly separates coarse initialization from fine-grained densification, ensuring global geometry is established before local details are added. It differs from standard training by using a mesh-based or point-based coarse stage followed by iterative densification triggered by error metrics (e.g., DSSIM). Choose this when high-fidelity reconstruction of complex geometry is required, particularly in mesh-aligned variants. See [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md).

### [Training Objective and Densification](7448-training-objective-and-densification.md)
Defines loss functions and densification protocols for **MeshGS**, integrating explicit mesh geometry into the optimization pipeline. It solves the problem of Gaussian primitives drifting from known surface geometry by anchoring "tightly-bound" splats to a mesh while retaining standard densification for "loosely-bound" splats to handle occlusions. Choose this when aligning Gaussian Splatting with explicit mesh representations. See [Training Objective and Densification](7448-training-objective-and-densification.md).

### [Fine-tuning](9138-fine-tuning.md)
Refers to the phase where a pre-initialized set of Gaussian primitives (often from a feed-forward network) is optimized for novel view synthesis. It sets itself apart by enabling growing and pruning during optimization to discard redundant initial Gaussians, accelerating convergence and reducing memory compared to training from scratch. Choose this when leveraging pretrained priors or sparse initializations. See [Fine-tuning](9138-fine-tuning.md).

### [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md)
Stabilizes optimization by alternating between sparse voxel sampling (global consistency) and structured slice-based sampling (local detail). It addresses the instability of random voxel sampling and the lack of global guidance in slice-only supervision. Choose this for volumetric data (e.g., MRI) where global structure and local detail must be balanced. See [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md).

### Progressive Propagation Strategy
Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage [S1, unofficial].

### Interleaved Densification
Alternates between spatial and temporal densification every 2 epochs to avoid Gaussians failing to reconstruct either motion or appearance detail [S2].

## Background
The concept of multi-stage training in 3D Gaussian Splatting emerged as a response to the limitations of single-stage optimization, which often struggles to balance global structure and local detail [S1, unofficial]. The Hybrid Coarse-to-Fine Strategy and MeshGS variants explicitly formalize this separation [S1, unofficial]. Progressive propagation strategies, such as those in GaussianPro, aim to smooth the expansion of the Gaussian cloud [S1, unofficial]. In dynamic scenes, Interleaved Densification addresses the difficulty of deciding between spatial and temporal densification by alternating them [S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
