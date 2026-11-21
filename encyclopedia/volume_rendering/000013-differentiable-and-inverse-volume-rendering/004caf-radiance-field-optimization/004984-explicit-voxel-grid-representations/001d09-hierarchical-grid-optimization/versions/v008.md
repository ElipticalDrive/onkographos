# Hierarchical Grid Optimization


## Overview
Hierarchical Grid Optimization is a family of differentiable volume rendering methods that optimize explicit 3D voxel grids (and related structures) to reconstruct radiance fields, solving the computational bottleneck of neural network inference in implicit representations like NeRF [S1, unofficial]. It sets itself apart from implicit methods by using direct data access and interpolation, and from static grid methods by dynamically adapting resolution or sparsity during training to balance fidelity and memory usage [S1, unofficial].

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses).

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [S1, unofficial][S5]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S1, unofficial][S5].

Differentiation strategies fall into three main categories:
1.  **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [S5].
2.  **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [S1, unofficial][S3].
3.  **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [S5].

## Subtopics

### [Explicit Radiance Fields](806-explicit-radiance-fields.md)
Explicit Radiance Fields (PERF) is a method for image-based 3D reconstruction that formulates volumetric reconstruction as a non-linear least-squares problem solved explicitly without neural networks. It uses a voxel grid for the foreground and a hierarchy of environment maps for the background, optimizing color and opacity values directly via a Gauss-Newton solver. Unlike [Plenoxels](7440-plenoxels.md) which uses neural networks for view-dependent color, PERF assumes diffuse surfaces and achieves significantly faster convergence by leveraging analytical gradients and iterative linear solvers.

### [Plenoxels Optimization Configuration](929-plenoxels-optimization-configuration.md)
Plenoxels Optimization Configuration defines the specific hyperparameter settings and training protocols required to optimize the voxel opacity ($\sigma$) and spherical harmonic (SH) coefficients in a Plenoxels scene representation. It solves the problem of efficiently converging to high-fidelity radiance fields by selecting appropriate gradient-based optimizers and learning rate schedules. It differs from general neural field optimization by relying on explicit grid structures and SH basis functions rather than MLPs, and from [PlenOctree Optimization](1291-plenoctree-optimization.md) by using a fixed-resolution voxel grid instead of a hierarchical octree structure.

### [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-optimization-with-pruning.md)
Coarse-to-Fine Optimization with Pruning is a hierarchical training strategy for 3D radiance fields represented as voxel grids. It solves the problem of efficiently initializing and refining high-resolution scene representations by starting with a coarse dense grid, pruning unoccupied regions, and iteratively subdividing the remaining voxels to higher resolutions. It sets itself apart from static grid methods by dynamically adapting the voxel structure to scene complexity during training, building on the general framework of [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md).

### [PlenOctree Optimization](1291-plenoctree-optimization.md)
PlenOctree Optimization is a post-training refinement process that finetunes a converted PlenOctree on original training images using Stochastic Gradient Descent (SGD) to improve image quality. It solves the residual approximation errors introduced during the conversion from a neural network to a sparse voxel grid. Unlike [Plenoxels Optimization Configuration](929-plenoxels-optimization-configuration.md), which configures fixed grid parameters, PlenOctree Optimization actively adjusts the density and spherical harmonic coefficients of the octree leaves. It also differs from [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-with-pruning.md) by operating on a pre-converted sparse structure rather than optimizing the network and sparsity simultaneously.

### [Sparse Voxel Grid Optimization](2568-sparse-voxel-grid-optimization.md)
Sparse Voxel Grid Optimization is a differentiable volume rendering method that optimizes a sparse 3D voxel grid for density and color features, enabling real-time high-fidelity radiance field reconstruction. It distinguishes itself from hierarchical methods like [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md) by avoiding complex multi-resolution tree structures, instead relying on efficient occupancy caching, Morton code sorting, and near-camera voxel masking to handle unbounded scenes and accelerate training. It builds on the standard emission-absorption model and differentiability principles defined in [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md).

### [Multi-scale Neural Field Reconstruction](7437-multi-scale-neural-field-reconstruction.md)
Multi-scale Neural Field Reconstruction is a framework that constructs pyramid-structured encoding volumes to provide geometric information at all scales for Neural Radiance Fields (NeRF), addressing the limitation of single-scale volumes in scenes with diverse object sizes. It learns an encoding volume and reconstructs density and radiance fields separately, using multi-scale geometric information acquired during training (rendering at each level) and used only at the final level during testing. It sets itself apart from siblings like [Explicit Radiance Fields](806-explicit-radiance-fields.md) by explicitly modeling scale via a pyramid of 3D volumes rather than a single grid or octree, and from [Plenoxels](7440-plenoxels.md) by retaining differentiable neural field reconstruction at each scale rather than relying on spherical harmonics or explicit voxel colors directly.

### [Plenoxels](7440-plenoxels.md)
Plenoxels is an explicit radiance field representation that replaces the neural network used in NeRF with a sparse 3D voxel grid storing density and spherical harmonic (SH) coefficients. It solves the computational bottleneck of neural network inference during rendering and optimization, enabling training speeds two orders of magnitude faster than NeRF and real-time rendering capabilities. Unlike implicit representations, Plenoxels relies on direct data access and trilinear interpolation, trading increased memory consumption for significant gains in efficiency.

#### Baking Neural Fields (SNeRG)
- **Definition:** Uses a hierarchical data structure: coarsest level is a $(N / B)^3$ resolution dense voxel grid pointing to blocks of a size-$B^3$ dense voxel grid, making use of sparsity while keeping data contiguous [S1, unofficial]. Decomposes an image into diffuse color (view-independent) and specularity (view-dependent) so the inference network handles a simple task [S1, unofficial].
- **Step:** Computes weighted sum of intermediate features and then evaluates color per pixel, significantly reducing the number of feed forward passes compared to evaluating color per 3D sample [S1, unofficial].
- **Pitfall:** Octrees' intermediate non-leaf nodes and tree traversals incur significant memory and time overhead, which this method avoids by using block pointers [S1, unofficial].

#### Density Voxel Grid Reconstruction
- **Definition:** A scene representation method using a density voxel grid to generate high-quality 3D reconstructed scenes and images once camera poses are optimized [S2].
- **Comparison:** Unlike NeRF which uses an MLP, or Plenoxels which uses sparse voxels with spherical harmonics, CBARF uses a density voxel grid for the final reconstruction stage after pose refinement [S2].

#### Canonical Radiance Field with Voxel Grid
- **Definition:** A representation of a static scene at canonical time using a learnable voxel radiance feature grid and a lightweight MLP, replacing the heavy MLP in NeRF to allow querying finite 3D points [S3].
- **Formula:** $\mathbf{V}_{\text{R}}^{\text{Can}} = \text{RadNet} (\mathbf{V}_{\text{p}}^{\text{Can}}, \mathbf{V}_{\text{Rf}}^{\text{Can}})$, where $\mathbf{V}_{\text{R}}^{\text{Can}} = \{V\sigma^{\text{Can}}, Vcf^{\text{Can}}\}$ consists of density voxel grid $V\sigma$ and color feature voxel grid $Vcf$. $Vp^{\text{Can}}$ are the 3D coordinates of the voxel grid in canonical space, embedded according to NeRF [26] [S3].
- **Parameter:** F$\theta_1$: A lightweight MLP network used to model the radiance [S3].

## Background
The concept of explicit voxel grids for radiance fields emerged as a solution to the slow training and inference speeds of implicit Neural Radiance Fields (NeRF). Plenoxels [S1, S2, S4, S5] was a pivotal work that demonstrated that replacing MLPs with sparse voxel grids storing density and spherical harmonic coefficients could achieve training speeds two orders of magnitude faster than NeRF. Subsequent works like DVGO [S2] and Sparse Voxel Grid Optimization [S1, unofficial] further refined this by introducing sparsity, Morton code sorting, and dynamic pruning to handle unbounded scenes and reduce memory usage. Hierarchical strategies, such as Coarse-to-Fine Optimization [S1, unofficial] and PlenOctree Optimization [S1, unofficial], were developed to manage the trade-off between resolution and computational cost by starting with coarse representations and refining them. Explicit Radiance Fields (PERF) [S1, unofficial] took a different approach by using analytical Gauss-Newton solvers instead of gradient descent, assuming diffuse surfaces to accelerate convergence.

## Sources
[S1] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S2] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
[S3] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
