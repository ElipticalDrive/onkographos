# Explicit Voxel/Grid Representations

Also called: Explicit Radiance Fields, Direct Voxel Grid Optimization, Voxel-based NeRF

## Overview
Explicit Voxel/Grid Representations optimize scene parameters (density, color, features) stored in discrete 3D grids or hierarchical structures, avoiding the implicit parameterization of multi-layer perceptrons (MLPs) used in [Neural Radiance Fields](4972-neural-radiance-fields.md). This family distinguishes itself by enabling direct optimization of grid values, which significantly reduces training time and memory overhead compared to implicit methods, while offering more structured scene representations than point-based methods like [3D Gaussian Splatting](1795-3d-gaussian-splatting.md).

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [44-radiance-field-optimization.md]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [44-radiance-field-optimization.md].

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

### [Explicit Radiance Fields](806-explicit-radiance-fields.md)
Explicit Radiance Fields (PERF) formulates image-based 3D reconstruction as a non-linear least-squares problem solved explicitly without neural networks. It represents the scene with a voxel grid for the foreground and a hierarchy of environment maps for the background, optimizing color and opacity values directly via a Gauss-Newton solver. Unlike [Plenoxels](7440-plenoxels.md) which uses neural networks for view-dependent color, PERF assumes diffuse surfaces and achieves significantly faster convergence by leveraging analytical gradients and iterative linear solvers.

### [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md)
Direct Voxel Grid Optimization (DVGO) is a method for reconstructing per-scene radiance fields from calibrated images by directly optimizing a dense 3D voxel grid for volume density and a feature voxel grid for color, achieving convergence in less than 15 minutes on a single GPU. It sets itself apart from implicit methods like [Neural Radiance Fields](4972-neural-radiance-fields.md) by eliminating the need for lengthy MLP training, and from earlier explicit voxel methods by training the voxel grid directly from scratch without requiring a conversion step from a pre-trained implicit model. Unlike [Plenoxels](806-explicit-radiance-fields.md) which optimizes colors directly in the grid, DVGO uses a shallow MLP to handle view-dependent appearance, balancing speed and quality.

### Canonical Radiance Field with Voxel Grid
A representation of a static scene at canonical time using a learnable voxel radiance feature grid and a lightweight MLP, replacing the heavy MLP in NeRF to allow querying finite 3D points [S1]. The canonical radiance field is defined as $\mathbf{R}_{\text{Can}} = \text{RadNet}(\mathbf{p}_{\text{Can}}, \mathbf{Rf}_{\text{Can}})$, where $\mathbf{R}_{\text{Can}} = \{V\sigma^{\text{Can}}, Vcf^{\text{Can}}\}$ consists of density voxel grid $V\sigma$ and color feature voxel grid $Vcf$. $Vp^{\text{Can}}$ are the 3D coordinates of the voxel grid in canonical space, embedded according to NeRF [26] [S1]. It uses a lightweight MLP network $F_{\theta1}$ to model the radiance [S1].

## Background
Explicit voxel representations emerged as a response to the computational bottlenecks of implicit neural radiance fields. Early works like [Explicit Radiance Fields](806-explicit-radiance-fields.md) demonstrated that direct optimization of voxel grids could achieve fast convergence for static scenes. [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md) later popularized the approach by showing that a dense voxel grid combined with a shallow MLP could match the quality of NeRF while training orders of magnitude faster. These methods are widely used in real-time rendering, dynamic scene reconstruction, and scenarios requiring rapid per-scene optimization [S1, S2, S4].

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
