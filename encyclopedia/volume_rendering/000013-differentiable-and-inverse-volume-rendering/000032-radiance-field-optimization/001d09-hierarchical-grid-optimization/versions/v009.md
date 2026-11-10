# Hierarchical Grid Optimization


## Overview
Hierarchical Grid Optimization is a family of methods that optimize explicit 3D voxel grids (and related structures like octrees) to reconstruct radiance fields, solving the computational bottleneck of implicit neural networks by replacing them with direct data access and interpolation [S1, unofficial][S3]. It sets itself apart from implicit methods like [Neural Radiance Fields](4972-neural-radiance-fields.md) by enabling training speeds two orders of magnitude faster than NeRF and real-time rendering capabilities, while distinguishing itself from static grid methods by dynamically adapting the voxel structure to scene complexity during training [S1, unofficial][S3].

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

### [Plenoxels Optimization Configuration](929-plenoxels-optimization-configuration.md)
Plenoxels Optimization Configuration defines the hyperparameter settings and training protocols required to optimize the voxel opacity ($\sigma$) and spherical harmonic (SH) coefficients of a sparse 3D grid representation. It solves the problem of efficiently converging to high-fidelity radiance fields without neural networks by selecting appropriate gradient-based optimizers and learning rate schedules. It is distinguished from sibling methods like [Explicit Radiance Fields](806-explicit-radiance-fields.md) and [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-optimization-with-pruning.md) by its specific reliance on RMSProp for SH coefficients and its robustness to learning rate schedule variations, while building on the differentiable volume rendering principles of [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md).

### [Coarse-to-Fine Optimization with Pruning](1153-coarse-to-fine-optimization-with-pruning.md)
Coarse-to-Fine Optimization with Pruning is a hierarchical training strategy for 3D radiance fields represented as voxel grids. It solves the problem of efficiently initializing and refining high-resolution scene representations by starting with a coarse dense grid, pruning unoccupied regions, and iteratively subdividing the remaining voxels to higher resolutions. It sets itself apart from static grid methods by dynamically adapting the voxel structure to scene complexity during training, building on the general framework of [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md).

### [PlenOctree Optimization](1291-plenoctree-optimization.md)
PlenOctree Optimization is a post-processing stage that refines a pre-converted PlenOctree representation by optimizing its leaf node parameters (density and spherical harmonics coefficients) against original training images using stochastic gradient descent (SGD). It solves the issue of approximation errors introduced during the initial conversion from a neural network to a sparse voxel grid, thereby improving final image quality without altering the tree structure. Unlike [PlenOctree Conversion](1359-plenoctree-conversion.md), which focuses on the construction and tabulation of the view-dependent volume, this method focuses on the differentiable optimization of the already-sparse structure. It is distinct from [Structure Refinement](929-plenoxels-optimization-configuration.md) in that it operates on a fixed tree topology, whereas the latter dynamically merges, splits, and culls nodes.

### [Sparse Voxel Grid Optimization](2568-sparse-voxel-grid-optimization.md)
Sparse Voxel Grid Optimization is a differentiable volume rendering method that optimizes a sparse 3D voxel grid for density and color features, enabling real-time high-fidelity radiance field reconstruction. It distinguishes itself from sibling methods by using an Octree structure with Morton code sorting and near-camera voxel masking to handle unbounded scenes and accelerate training. It builds on the standard emission-absorption model and differentiability principles defined in [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md).

### [Direct Voxel Grid Optimization](4694-direct-voxel-grid-optimization.md)
Direct Voxel Grid Optimization (DVGO) is a method for reconstructing per-scene radiance fields from calibrated images by directly optimizing a dense 3D voxel grid for volume density and a feature voxel grid for color, achieving convergence in less than 15 minutes on a single GPU. It sets itself apart from implicit methods like [Neural Radiance Fields](4972-neural-radiance-fields.md) by replacing the large spatial MLP with explicit voxel grids, and from previous explicit voxel methods by training the grid directly from scratch without requiring a conversion step from a pre-trained implicit model. This direct optimization approach reduces training time from hours/days to minutes while maintaining or exceeding the visual quality of NeRF.

### [Multi-scale Neural Field Reconstruction](7437-multi-scale-neural-field-reconstruction.md)
Multi-scale Neural Field Reconstruction is a framework that constructs pyramid-structured encoding volumes to provide geometric information at all scales for Neural Radiance Fields (NeRF), addressing the limitation of single-scale volumes in scenes with diverse object sizes. It learns an encoding volume and reconstructs density and radiance fields separately, using multi-scale geometric information acquired during training (rendering at each level) and used only at the final level during testing. It sets itself apart from siblings like [Explicit Radiance Fields](806-explicit-radiance-fields.md) by explicitly modeling scale via a pyramid of 3D volumes rather than a single grid or octree, and from [Plenoxels](7440-plenoxels.md) by retaining differentiable neural field reconstruction at each scale rather than relying on spherical harmonics or explicit voxel colors directly.

### [Plenoxels](7440-plenoxels.md)
Plenoxels is an explicit radiance field representation that replaces the neural network used in NeRF with a sparse 3D voxel grid storing density and spherical harmonic (SH) coefficients. It solves the computational bottleneck of neural network inference during rendering and optimization, enabling training speeds two orders of magnitude faster than NeRF and real-time rendering capabilities. Unlike implicit representations, Plenoxels relies on direct data access and trilinear interpolation, trading increased memory consumption for significant gains in efficiency.

### [HDR-Plenoxels](12123-hdr-plenoxels.md)
HDR-Plenoxels is a voxel-based volume rendering method that reconstructs 3D High Dynamic Range (HDR) radiance fields, scene geometry, and camera settings from multi-view Low Dynamic Range (LDR) images captured under varying exposure, white balance, and camera response functions (CRF). It extends Plenoxels by introducing a differentiable tone-mapping module that models the in-camera imaging pipeline, allowing the network to learn HDR radiance and self-calibrate camera parameters end-to-end. Unlike implicit neural fields, it uses explicit sparse voxel grids with spherical harmonics, achieving rendering speeds two orders of magnitude faster than NeRF while handling saturated regions and extreme exposure conditions robustly.

#### Baking Neural Fields (SNeRG)
- **Definition:** Uses a hierarchical data structure: coarsest level is a $(N / B)^3$ resolution dense voxel grid pointing to blocks of a size-$B^3$ dense voxel grid, making use of sparsity while keeping data contiguous [S1, unofficial]. Decomposes an image into diffuse color (view-independent) and specularity (view-dependent) so the inference network handles a simple task [S1, unofficial].
- **Step:** Computes weighted sum of intermediate features and then evaluates color per pixel, significantly reducing the number of feed forward passes compared to evaluating color per 3D sample [S1, unofficial].
- **Pitfall:** Octrees' intermediate non-leaf nodes and tree traversals incur significant memory and time overhead, which this method avoids by using block pointers [S1, unofficial].

#### Density Voxel Grid Reconstruction
- **Definition:** A scene representation method using a density voxel grid to generate high-quality 3D reconstructed scenes and images once camera poses are optimized [S2].
- **Comparison:** Unlike NeRF which uses an MLP, or Plenoxels which uses sparse voxels with spherical harmonics, CBARF uses a density voxel grid for the final reconstruction stage after pose refinement [S2].

#### Canonical Radiance Field with Voxel Grid
- **Definition:** A representation of a static scene at canonical time using a learnable voxel radiance feature grid and a lightweight MLP, replacing the heavy MLP in NeRF to allow querying finite 3D points [S3].
- **Formula:** $\Vgrid {R}{Can} = \RadNet (\Vgrid {p}{Can}, \Vgrid {Rf}{Can})$, where $\Vgrid {R}{Can} = \{V\sigma^{Can}, Vcf^{Can}\}$ consists of density voxel grid $V\sigma$ and color feature voxel grid $Vcf$. $Vp^{Can}$ are the 3D coordinates of the voxel grid in canonical space, embedded according to NeRF [26] [S3].
- **Parameter:** Fθ1: A lightweight MLP network used to model the radiance [S3].

## Background
Hierarchical Grid Optimization originated as a response to the high computational cost of implicit neural representations like NeRF. Direct Voxel Grid Optimization (DVGO) demonstrated that explicit voxel grids could be optimized directly from scratch, reducing training time from hours to minutes [S1, unofficial][S3]. Plenoxels further accelerated this by replacing MLPs with spherical harmonics, enabling real-time rendering [S1, unofficial][S2]. The family has evolved to include sparse structures (PlenOctree, Sparse Voxel Grid) to handle unbounded scenes and multi-scale representations to capture diverse object sizes [S1, unofficial][S3]. It is widely used in real-time novel view synthesis, 3D reconstruction from images, and HDR scene reconstruction [S1, unofficial][S2, S3].

## Sources
[S1] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S2] CBARF: Cascaded Bundle-Adjusting Neural Radiance Fields From Imperfect Camera Poses, https://doi.org/10.1109/tmm.2024.3388929, fetched 2026-10-26
[S3] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
