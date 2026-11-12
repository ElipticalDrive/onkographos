# Progressive and Multi-stage Training

Also called: Progressive Training, Multi-stage Optimization, Curriculum Training for 3DGS

## Overview
Progressive and Multi-stage Training refers to methodologies that optimize 3D Gaussian Splatting representations through distinct phases or stages rather than a single monolithic optimization loop [S1, unofficial]. It solves the problem of converging to local minima, producing blurry details, or suffering from instability when initializing with sparse or noisy data, by structuring the learning process to first establish coarse global structure and then refine local details [S1, unofficial]. It is distinguished from single-stage training by its iterative densification and curriculum-based supervision, which yields higher rendering quality and stability, and from pure mesh-based approaches by its focus on point cloud attribute optimization within a differentiable rendering framework [S1, 7173-hybrid-coarse-to-fine-strategy.md, unofficial].

## Shared principles
The core mechanism of progressive and multi-stage training involves structuring the optimization timeline into discrete phases, each with specific objectives, constraints, or supervision signals. Common principles include:

1.  **Phase Segmentation:** The training process is divided into distinct stages (e.g., coarse initialization, fine-grained densification, fine-tuning) [7173-hybrid-coarse-to-fine-strategy.md, 9138-fine-tuning.md]. Transitions between stages are often triggered by iteration counts or convergence metrics.
2.  **Curriculum Learning:** The difficulty or type of supervision signal changes over time. For example, methods may start with global or coarse constraints (e.g., sparse points, slice supervision) and progressively introduce finer details or denser constraints (e.g., voxel supervision, high-frequency regularization) [12921-curriculum-learning-with-slice-and-voxel-supervision.md].
3.  **Staged Densification:** Densification (splitting/cloning) is not performed uniformly throughout training. It may be restricted to specific phases (e.g., only during the coarse-to-fine densification phase) or enabled only in later stages (e.g., fine-tuning) to manage redundancy and memory [7173-hybrid-coarse-to-fine-strategy.md, 9138-fine-tuning.md].
4.  **Loss Function Evolution:** The loss function components may be weighted or introduced sequentially. For instance, geometric consistency losses might be emphasized in early stages, while photometric losses dominate later [7448-training-objective-and-densification.md, 12921-curriculum-learning-with-slice-and-voxel-supervision.md].

## Subtopics

### [Hybrid Coarse-to-Fine Strategy](7173-hybrid-coarse-to-fine-strategy.md)
The Hybrid Coarse-to-Fine Strategy refines geometric density through multiple stages, starting with a coarse initialization and progressing to fine-grained densification [7173-hybrid-coarse-to-fine-strategy.md]. It is distinguished from single-stage training by its iterative densification loop, which yields significantly higher rendering quality, and from pure mesh-based approaches by its focus on point cloud attribute optimization within a differentiable rendering framework. It operates in two phases:
1.  **Coarse Initialization:** Edge-aware sampling on downsampled images, Delaunay triangulation, and attribute initialization [7173-hybrid-coarse-to-fine-strategy.md].
2.  **Iterative Densification:** Periodic scoring of triangles based on mean DSSIM and area ($\text{score}_t = \overline{\text{DSSIM}}_t \cdot |\text{pixels}_t|^{0.75}$), insertion of centroids, and re-meshing [7173-hybrid-coarse-to-fine-strategy.md].
Densification runs every 500 iterations from step 1,500 to 15,000, capping new vertices at 35% of the current count [7173-hybrid-coarse-to-fine-strategy.md].

### [Training Objective and Densification](7448-training-objective-and-densification.md)
This method defines loss functions and densification protocols for **MeshGS**, aligning primitives with an explicit mesh surface [7448-training-objective-and-densification.md]. It solves the problem of Gaussian splats drifting away from geometry by splitting primitives into **tightly-bound** ($G^t$) and **loosely-bound** ($G^l$) sets [7448-training-objective-and-densification.md]. $G^t$ splats are constrained by mesh-specific losses (normal consistency, scale regularization, projection loss) and densified by sampling from mesh triangles, while $G^l$ splats follow standard 3DGS densification [7448-training-objective-and-densification.md].

### [Fine-tuning](9138-fine-tuning.md)
Fine-tuning is the final optimization stage in the 3DGS pipeline, designed to refine geometry, color, and opacity of a pre-initialized set of Gaussians [9138-fine-tuning.md]. It solves the problem of slow convergence and high memory usage by leveraging pretrained priors or feed-forward estimates [9138-fine-tuning.md]. It sets itself apart from earlier stages by enabling active growing and pruning of primitives to eliminate redundancy inherent in initial estimates, a capability not typically utilized during the initial densification phase [9138-fine-tuning.md]. It uses the vanilla 3DGS loss function with Adam optimizer [9138-fine-tuning.md].

### [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md)
This method addresses instability in shear-warp rendering by enforcing a structured training curriculum [12921-curriculum-learning-with-slice-and-voxel-supervision.md]. It distinguishes itself from sibling methods by explicitly alternating between planar slice constraints and global voxel constraints, ensuring the Gaussian representation learns both local texture continuity and global spatial consistency [12921-curriculum-learning-with-slice-and-voxel-supervision.md]. It leverages the geometric alignment between slice-based supervision and the shear-warp rendering pipeline to stabilize optimization [12921-curriculum-learning-with-slice-and-voxel-supervision.md].

### Progressive Propagation Strategy
Introduces a progressive propagation strategy to expand the Gaussian cloud, ensuring smooth expansion and better coverage [S1, unofficial]. It expands the Gaussian cloud in a fronto-parallel manner [S1, unofficial].

### Interleaved Densification
Alternates between spatial and temporal densification every 2 epochs to avoid Gaussians failing to reconstruct either motion or appearance detail [S2].

## Background
Progressive and multi-stage training strategies have emerged as essential components of 3D Gaussian Splatting optimization to address the limitations of single-stage training, such as local minima and instability. The Hybrid Coarse-to-Fine Strategy was introduced to improve detail reconstruction by iteratively densifying high-error regions [7173-hybrid-coarse-to-fine-strategy.md]. MeshGS introduced stage-specific losses and densification to align Gaussians with explicit geometry [7448-training-objective-and-densification.md]. Fine-tuning protocols were developed to leverage feed-forward initialization for faster convergence [9138-fine-tuning.md]. Curriculum learning approaches, such as slice and voxel supervision, were proposed to stabilize volumetric reconstruction in medical imaging contexts [12921-curriculum-learning-with-slice-and-voxel-supervision.md]. Progressive propagation and interleaved densification address specific challenges in coverage and spatio-temporal reconstruction, respectively [S1, unofficial][S2].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Adaptive Spatio‐Temporal 3D Gaussian Splatting for Scenes with Oscillatory Motion, https://doi.org/10.1111/cgf.70410, fetched 2026-10-26
