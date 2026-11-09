# Voxel-based Representation

Also called: Voxel Grids, Explicit Voxel Representations

## Overview
Voxel-based representations replace the continuous multi-layer perceptron (MLP) of standard Neural Radiance Fields with discrete 3D voxel grids to store scene features, density, or radiance values [S1][S2, unofficial]. This approach solves the high computational cost and memory overhead of evaluating neural networks for every sample along a ray, enabling significantly faster rendering speeds [S1][S2, unofficial]. It sets itself apart from sibling methods like [NeRF](1185-nerf-neural-radiance-fields.md) by trading model compactness for rendering speed via explicit spatial caching and lookup operations [S1][S2, unofficial].

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses).

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [S1][S5]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S1][S5].

Differentiation strategies fall into three main categories:
1.  **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [S5].
2.  **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [S1][S3, unofficial].
3.  **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [S5].

## Subtopics

### [FastNeRF](7183-fastnerf.md)
FastNeRF accelerates Neural Radiance Fields by caching view-independent radiance values at selected 3D points sorted in a sparse spatial structure, such as voxel grid vertices. It solves the high memory and computational cost of evaluating a full MLP for every sample along a ray. Unlike standard NeRF, which evaluates the network on the fly, FastNeRF precomputes and stores these values, enabling high-fidelity photorealistic rendering at 200fps on high-end consumer GPUs. It sets itself apart from siblings like V4D and RenderNet by explicitly trading model compactness for rendering speed via explicit spatial caching.

### [V4D (Voxel for 4D Novel View Synthesis)](8762-v4d-voxel-for-4d-novel-view-synthesis.md)
V4D is a voxel-based architecture for 4D novel view synthesis in dynamic scenes that models the 4D neural radiance field directly without requiring a canonical space. It addresses the limitations of MLP-only representations by using separate 3D voxel grids for density and texture, combined with a conditional positional encoding (CPE) module and a look-up table (LUT) refinement module to recover high-frequency details lost by total variation regularization. Unlike canonical-space methods such as [D-NeRF](7274-dynerf.md), V4D maps 6D attributes (3D spatial location, time, and view direction) directly to volume properties, trading longer training times for competitive inference speeds and state-of-the-art performance on single-view dynamic datasets.

### [RenderNet](10878-rendernet.md)
A differentiable rendering convolutional neural network that renders 2D images from 3D voxel grids by jointly learning visibility computation and pixel color calculation (shading).

### C-NeRF
#### definition
- [S1] A baseline method that adopts a 3D CNN to produce feature volumes and appends a NeRF-like MLP to predict radiance fields.
#### comparison
- [S1] Tends to lose details in high frequency regions of the image because of its single shared MLP model structure.

## Background
Voxel-based representations in the context of neural radiance fields emerged as a solution to the computational bottlenecks of MLP-based NeRFs. FastNeRF introduced the concept of caching radiance values in sparse voxel grids to achieve real-time rendering speeds [S1][S2, unofficial]. V4D extended voxel-based approaches to dynamic 4D scenes, demonstrating that explicit voxel grids could outperform canonical-space MLP methods when combined with refinement modules [S1]. C-NeRF served as an early baseline, combining 3D CNN feature volumes with MLPs, though it suffered from detail loss in high-frequency regions [S1]. These methods collectively established the viability of explicit volumetric representations for efficient and high-quality novel view synthesis.

## Sources
[S1] Representing Volumetric Videos as Dynamic MLP Maps, https://arxiv.org/abs/2304.06717, fetched 2026-10-29
