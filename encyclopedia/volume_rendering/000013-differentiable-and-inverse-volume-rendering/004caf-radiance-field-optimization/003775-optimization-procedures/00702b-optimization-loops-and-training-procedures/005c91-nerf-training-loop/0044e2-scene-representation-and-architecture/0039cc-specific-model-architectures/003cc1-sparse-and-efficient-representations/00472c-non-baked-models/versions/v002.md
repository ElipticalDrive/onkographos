# Non-Baked Models

Also called: Hybrid Representations, Learning separate scene features from MLP parameters

## Overview
Non-baked models learn separate scene features from MLP parameters via hybrid representations, enabling smaller MLPs for improved training and inference speed at the cost of increased memory [S1]. Unlike baked models that pre-compute and store final MLP outputs to accelerate inference without affecting training time, non-baked models retain the differentiability of the feature-to-output mapping during optimization, improving both phases [S1].

## Shared principles

## Subtopics

### [AdaNeRF](6195-adanerf.md)
Adapts the representation dynamically.

### [SinNeRF](6199-sinnerf.md)
Uses sinusoidal activations.

### [CLIP-NeRF](8661-clip-nerf.md)
Integrates CLIP embeddings.

### [NeRF-RPN](8663-nerf-rpn.md)
Uses region proposal networks.

### [RefSR-NeRF](8691-refsr-nerf.md)
Focuses on reflection and super-resolution.

### [AligNeRF](8695-alignerf.md)
Improves alignment.

### [DiffusioNeRF](8703-diffusionerf.md)
Integrates diffusion models.

### [Pool proposer architecture](8953-pool-proposer-architecture.md)
Uses pooling mechanisms.

### [MS-NeRF](9111-ms-nerf.md)
Multi-scale representations.

### [NeRDi](9112-nerdi.md)
Depth-integrated representation.

### [RING-NeRF](9323-ring-nerf.md)
Ring-based acquisition handling.

### [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md)
Optimizes sparsity.

### [Edit-NeRF](9388-edit-nerf.md)
Enables scene editing.

### [Geometric Priors and Depth Integration](9395-geometric-priors-and-depth-integration.md)
Uses depth priors.

### [Generative and Style-based Models](9572-generative-and-style-based-models.md)
Generative approaches.

### [IRON](10251-iron.md)
Implicit Octree Radiance Fields.

### [Instance-NeRF](10611-instance-nerf.md)
Instance-specific fields.

### [MIMO-NeRF](10928-mimo-nerf.md)
Multi-input multi-output.

### [Appearance and Lighting Decomposition](11061-appearance-and-lighting-decomposition.md)
Separates lighting.

### [STensoRRF](11359-stensorrf.md)
Tensor-based radiance fields.

### [Ced-NeRF](11364-ced-nerf.md)
Conditional editing.

### [Dynamic and Deformable Fields](11366-dynamic-and-deformable-fields.md)
Handles dynamic scenes.

## Background
The classification originates from distinguishing NeRF optimization procedures into baked models (pre-computed outputs) and non-baked models (learning separate scene features from MLP parameters) [S1]. Non-baked models emerged to mitigate memory bottlenecks in standard MLP-based NeRFs by reducing network size, which improves training and inference speed while increasing memory overhead for explicit feature storage [S1]. The category encompasses hybrid representations and, pushed further, purely explicit scene representations that omit neural networks entirely; these are included due to their relevance and similarity to NeRF [S1]. Common techniques associated with this family include ray termination, space skipping, and hierarchical sampling using coarse and fine MLPs [S1].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
