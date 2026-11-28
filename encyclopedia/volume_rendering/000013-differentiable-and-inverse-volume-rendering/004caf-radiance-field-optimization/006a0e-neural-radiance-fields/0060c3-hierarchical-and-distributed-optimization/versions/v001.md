# Hierarchical and Distributed Optimization


## Overview
Hierarchical and Distributed Optimization refers to strategies for scaling Neural Radiance Field (NeRF) training to large-scale scenes or correcting imperfect camera poses by decomposing the optimization problem. It addresses the limitations of standard single-stage optimization, which fails with large initial pose errors [8223-cascaded-bundle-adjusting-neural-radiance-fields-cbarf.md] or suffers from memory bottlenecks and blending artifacts when scaling to arbitrarily large scenes [10947-nerf-xl-joint-training.md]. It sets itself apart from sibling methods like BlockNeRF or MegaNeRF by either using a cascaded coarse-to-fine pose refinement strategy [8223-cascaded-bundle-adjusting-neural-radiance-fields-cbarf.md] or by enforcing 0% overlap with joint training of global embeddings to eliminate train-test discrepancies [10947-nerf-xl-joint-training.md].

## Shared principles
The family relies on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [44-radiance-field-optimization.md]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [44-radiance-field-optimization.md].

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [44-radiance-field-optimization.md]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [44-radiance-field-optimization.md].

Differentiation strategies fall into three main categories [44-radiance-field-optimization.md]:
1. **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [44-radiance-field-optimization.md].
2. **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [44-radiance-field-optimization.md].
3. **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [44-radiance-field-optimization.md].

## Subtopics

### [Cascaded Bundle-Adjusting Neural Radiance Fields (CBARF)](8223-cascaded-bundle-adjusting-neural-radiance-fields-cbarf.md)
CBARF is a 3D reconstruction framework that simultaneously optimizes camera poses and scene representation from input images with imperfect or partially unknown camera poses [8223-cascaded-bundle-adjusting-neural-radiance-fields-cbarf.md]. It addresses the limitation where standard NeRF and single-stage Bundle Adjustment (BA) methods fail or over-fit when initial pose deviations are large [8223-cascaded-bundle-adjusting-neural-radiance-fields-cbarf.md]. Unlike single-stage BA which quickly falls into sub-optimal solutions, CBARF uses a coarse-to-fine multi-stage approach with cascaded compact BA modules to progressively refine poses, avoiding the learning of incorrect scene representations [8223-cascaded-bundle-adjusting-neural-radiance-fields-cbarf.md].

### [NeRF-XL Joint Training](10947-nerf-xl-joint-training.md)
NeRF-XL is a method for distributing Neural Radiance Fields across multiple GPUs by allocating NeRF parameters across a disjoint set of spatial regions and training them jointly, enabling arbitrarily large capacity without the redundancy and blending artifacts of independent training [10947-nerf-xl-joint-training.md]. Unlike sibling methods such as BlockNeRF and MegaNeRF which rely on overlapping regions and post-hoc blending, NeRF-XL uses 0% overlap and a single unified optimization process, eliminating train-test discrepancies and per-camera embedding inconsistencies [10947-nerf-xl-joint-training.md].

### [Low-Rank Constrained NeRF (LR-NeRF)](17322-low-rank-constrained-nerf-lr-nerf.md)
An optimization step where the simplified NeRF weights are finetuned under a low-rank constraint using the Alternating Direction Method of Multipliers (ADMM) to prepare for Tensor Train decomposition.

## Background
Not given in the sources.

## Sources
