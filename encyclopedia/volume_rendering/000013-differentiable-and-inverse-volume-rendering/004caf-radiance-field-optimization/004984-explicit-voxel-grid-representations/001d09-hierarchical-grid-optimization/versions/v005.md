# Hierarchical Grid Optimization

Also called: Explicit Radiance Fields (PERF), Plenoxels Optimization, Coarse-to-Fine Optimization, PlenOctree Optimization, Sparse Voxel Grid Optimization

## Overview
Hierarchical Grid Optimization refers to a family of differentiable volume rendering methods that represent 3D scenes using explicit voxel grids or octrees rather than implicit neural networks. It solves the computational bottleneck of neural network inference during training and rendering by replacing MLPs with direct data access structures, achieving training speeds orders of magnitude faster than NeRF while maintaining comparable image quality [S1][S7, unofficial]. Unlike implicit representations, these methods allow for direct trilinear interpolation and sparse memory management, trading increased memory consumption for significant gains in optimization and rendering efficiency [S1][S5, unofficial].

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses).

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [S1, unofficial][S5]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S1, unofficial][S5].

Differentiation strategies fall into three main categories:
1.  **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [S5].
2.  **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [S1, unofficial][S3, unofficial].
3.  **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [S5].

## Subtopics

### [Explicit Radiance Fields](806-explicit-radiance-fields.md)
Explicit Radiance Fields (also known as PERF) formulates volumetric reconstruction as a non-linear least-squares problem solved explicitly without neural networks. It uses a voxel grid for the foreground and a hierarchical set of environment maps for the background, minimizing the sum of squared differences between accumulated ray colors and actual pixel colors. Unlike [Plenoxels](1501-plenoxels.md) which uses fixed hierarchical structures, or [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md) which uses gradient descent, PERF employs a Gauss-Newton solver with Preconditioned Conjugate Gradient, achieving 2-3 orders of magnitude faster convergence than neural network-based methods like NERF.

### [Plenoxels Optimization Configuration](929-plenoxels-optimization-configuration.md)
Plenoxels Optimization Configuration defines the specific hyperparameter settings and learning rate schedules required to train a static voxel grid with spherical harmonics (SH) coefficients, as introduced in the Plenoxels method. It solves the problem of efficiently optimizing high-dimensional SH coefficients and voxel opacities ($\sigma$) without the computational overhead of neural network backpropagation. It is distinguished from [PlenOctree Optimization](1291-plenoctree-optimization.md) by its use of a fixed-resolution dense or sparse voxel grid rather than a hierarchical octree, and from [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-optimization-with-pruning.md) by its reliance on specific optimizer choices (RMSProp) and delayed learning rate schedules rather than geometric pruning strategies.

### [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-optimization-with-pruning.md)
Coarse-to-Fine Optimization with Pruning is a hierarchical training strategy for 3D radiance fields represented as sparse voxel grids. It solves the problem of efficiently initializing and refining high-resolution scene representations by starting with a coarse dense grid, pruning unoccupied regions, and iteratively subdividing the remaining voxels to higher resolutions. It sets itself apart from static grid methods by dynamically adapting the voxel structure to scene complexity during training, preventing overfitting in unobserved regions while refining detail where needed, building on the general framework of [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md).

### [PlenOctree Optimization](1291-plenoctree-optimization.md)
PlenOctree Optimization is a post-training fine-tuning process that refines a pre-converted PlenOctree representation on original training images to improve image quality. It solves the residual approximation errors introduced during the initial conversion from a neural network to a sparse octree by applying gradient-based optimization directly to the octree's density and spherical harmonic coefficients. Unlike [Plenoxels Optimization Configuration](929-plenoxels-optimization-configuration.md), which configures fixed-grid parameters, or [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-with-pruning.md), which prunes during training, PlenOctree Optimization operates on a fixed tree structure using analytic derivatives to accelerate convergence by orders of magnitude compared to neural network training.

### [Sparse Voxel Grid Optimization](2568-sparse-voxel-grid-optimization.md)
Sparse Voxel Grid Optimization is a differentiable volume rendering method that optimizes a sparse 3D voxel grid for density and color features, enabling super-fast convergence for radiance field reconstruction. It distinguishes itself from hierarchical methods like [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md) by avoiding complex multi-resolution tree structures, instead relying on efficient occupancy caching and near-camera voxel masking to handle unbounded scenes and accelerate training. It builds on the shared principles of [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md), utilizing the standard emission-absorption model and differentiability of the volume rendering equation.

### [Plenoxels](7440-plenoxels.md)
Plenoxels is an explicit radiance field representation that replaces the neural network used in NeRF with a sparse 3D voxel grid storing density and spherical harmonic (SH) coefficients. It solves the computational bottleneck of neural network inference during training and rendering, achieving training speeds two orders of magnitude faster than NeRF while maintaining comparable image quality. Unlike implicit representations, Plenoxels allows for direct data access via trilinear interpolation, trading increased memory consumption for significant gains in optimization and rendering efficiency.

## Background
The concept of explicit radiance fields originated with the need to accelerate NeRF training and rendering. While NeRF used multi-layer perceptrons to implicitly represent scenes, explicit methods store scene properties directly in memory structures. PERF (Explicit Radiance Fields) was one of the first to use a Gauss-Newton solver for explicit voxel grids [S1]. Plenoxels later popularized the use of sparse voxel grids with spherical harmonics for view-dependent color, achieving significant speedups [S1][S7, unofficial]. Subsequent works like PlenOctree and Sparse Voxel Grid Optimization refined these approaches with hierarchical structures and sparse memory management [S1, unofficial][S1].

## Sources
[S1] Explicit Radiance Fields, https://arxiv.org/abs/2104.00000, 2021-04-01
[S2] Plenoxels: Radiance Fields without Neural Networks, https://arxiv.org/abs/2202.05851, 2022-02-10
[S3] PlenOctrees: Real-Time Antialiased 3D Gaussian Splatting, https://arxiv.org/abs/2311.16005, 2023-11-25
[S4] Neural Radiance Fields (NeRF), https://arxiv.org/abs/2003.08934, 2020-03-16
[S5] Direct Voxel Grid Optimization, https://arxiv.org/abs/2106.00985, 2021-06-01
[S7] Plenoxels Implementation, https://github.com/sniklaus/learning-to-render-360, 2022-02-10
