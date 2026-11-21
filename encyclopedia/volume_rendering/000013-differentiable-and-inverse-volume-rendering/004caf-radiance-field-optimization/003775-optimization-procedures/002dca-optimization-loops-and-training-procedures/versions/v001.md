# Optimization Loops and Training Procedures

Also called: Radiance-field training loops, Neural volume rendering optimization

## Overview
This family defines the iterative procedures used to minimize the photometric error between rendered images and ground-truth observations by updating the parameters of differentiable volume or radiance fields. It differs from the underlying field representations (e.g., [Neural Radiance Fields](4972-neural-radiance-fields.md), [Plenoxels](5942-plenoxels.md)) by focusing on the optimization dynamics—specifically how rays are sampled, how gradients are propagated, and how hyperparameters like learning rates and regularization are scheduled.

## Shared principles
All methods in this family share the core objective of minimizing a photometric loss $\mathcal{L}$ between the rendered color $\hat{C}(\mathbf{r})$ and the ground-truth color $C(\mathbf{r})$ for a batch of rays $\mathcal{R}$:

$$
\mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \left\| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \right\|_2^2
$$

The optimization loop generally follows these steps:
1.  **Ray Sampling**: Random pixels are selected from training images. Ray origins and directions are computed using camera intrinsics ($K$) and extrinsics ($c2w$).
2.  **Forward Pass**: Points are sampled along rays within near/far bounds. The field model predicts density ($\sigma$) and color ($c$). Differentiable volume rendering computes the final pixel color.
3.  **Loss Computation**: The Mean Squared Error (MSE) is calculated. Additional terms (e.g., Eikonal, TV) may be added.
4.  **Backpropagation**: Gradients are computed and weights are updated via an optimizer (typically Adam).
5.  **Scheduling**: Learning rates and other hyperparameters are adjusted based on the current step or epoch.

## Subtopics

### [NeRF Training Loop](970-nerf-training-loop.md)
The standard training loop for coordinate-based neural fields (MLP or SIREN). It iterates over epochs, sampling random pixels, generating ray bundles, performing forward passes, and updating weights via backpropagation. It supports features like perturbative sampling, view-dependent encoding, and checkpointing. Choose this for standard NeRF-style reconstruction.

### [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md)
A specific hyperparameter configuration combining Adam with staircase exponential decay and selective L2 regularization. It uses a learning rate schedule that holds the initial rate for 10,000 steps, then decays by 1/3 every 10,000 steps. Crucially, it applies L2 weight decay ($10^{-6}$) only to MLP components, excluding positional encodings and appearance embeddings to prevent divergence. Choose this when high numerical stability and controlled regularization of embeddings are required.

### [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md)
Focuses on the basic MSE minimization for neural volume rendering. It serves as the foundational loss computation step for many other methods. Choose this for simple volumetric field training without complex regularization or stage-based strategies.

### [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md)
Alternates between image-based and ray-based sampling modes, applies stochastic preconditioning with exponential alpha decay, and optionally upsamples parameters between stages. Choose this for large-scale scenes or hierarchical representations where multi-resolution training is beneficial.

### [Staggered Optimization Scheme](5832-staggered-optimization-scheme.md)
Decouples the optimization of shape and Point Spread Function (PSF) into two sequential sub-problems. Choose this when dealing with systems where optical blur or PSF estimation is coupled with geometry reconstruction, to manage computational cost and bias.

### [Radiometric Self-Calibration](1820-radiometric-self-calibration.md)
Optimizes radiometric parameters (e.g., exposure, tone mapping) alongside the scene representation. Choose this for real-world datasets with varying lighting conditions or unknown camera response functions.

### [TensoRF](3026-tensorf.md)
Uses tensor decomposition for efficient optimization of high-dimensional fields. Choose this for memory-constrained environments or when faster convergence via low-rank approximations is needed.

### [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md)
Optimizes 3D Gaussians (position, covariance, color, opacity) using rasterization-based differentiable rendering. Choose this for real-time rendering applications and explicit surface representations.

### [Refractive Index Tomography using Neural Fields](4146-refractive-index-tomography-using-neural-fields.md)
Optimizes neural fields to represent refractive index distributions. Choose this for scientific imaging and tomography applications involving transparent media.

### [Neural Transient Fields (NeTF)](4760-neural-transient-fields-netf.md)
Extends optimization to 4D (3D space + time) for dynamic scenes. Choose this for capturing and rendering time-varying phenomena.

### [HybridNeRF](6363-hybridnerf.md)
Combines neural fields with explicit geometry (e.g., meshes) for optimization. Choose this when hybrid representations offer better convergence or rendering quality for specific scene types.

### [Color Volume Optimization and Density Initialization](842-color-volume-optimization-and-density-initialization.md)
Focuses on initializing density fields and optimizing color volumes. Choose this for methods that rely on explicit volume grids rather than implicit neural fields.

### [Neural Volumes](912-neural-volumes.md)
Optimizes neural volumes using 3D convolutions. Choose this for grid-based neural representations.

### [Best Viewpoint Selection via Opacity Entropy](1121-best-viewpoint-selection-via-opacity-entropy.md)
Uses opacity entropy to select optimal viewpoints for training. Choose this for active view synthesis or sparse view reconstruction.

### [Regularization losses](1413-regularization-losses.md)
Provides various regularization terms (e.g., TV, Eikonal) used in conjunction with the main photometric loss. Choose this to improve generalization or enforce physical constraints.

### [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md)
Optimizes hierarchical grid structures (e.g., hash grids) for efficient training. Choose this for large-scale scenes requiring multi-resolution representations.

### [Volumetric Inverse Rendering with Differentiable Path Tracing](1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md)
Uses differentiable path tracing for inverse rendering in volumetric scenes. Choose this for complex lighting and scattering effects.

### [3D Gaussian Splatting](1795-3d-gaussian-splatting.md)
The underlying representation for 3DGS training. Choose this as the base method for Gaussian-based rendering.

### [Neural Radiance Fields](4972-neural-radiance-fields.md)
The foundational neural field representation. Choose this as the base method for MLP-based rendering.

### [Plenoxels](5942-plenoxels.md)
The underlying representation for Plenoxels. Choose this as the base method for voxel-based rendering.

### [NeRF Training Loop](970-nerf-training-loop.md)
See above.

### [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md)
See above.

### [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md)
See above.

### [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md)
See above.

### [Staggered Optimization Scheme](5832-staggered-optimization-scheme.md)
See above.

### Photometric Optimization
Jointly optimizes feature grids $G_\sigma, G_a$, decoding MLP $f_{\text{MLP}}$, and environment map $E$. For real-world datasets with imperfect camera poses, a TV loss on the feature grid is additionally optimized to reduce noise. The optimization minimizes the photometric loss $\mathcal{L} = \frac{1}{|\mathcal{R}|} \sum_{\mathbf{r} \in \mathcal{R}} \| \hat{C}(\mathbf{r}) - C(\mathbf{r}) \|_2^2$. EgoNeRF does not need a coarse-to-fine reconstruction approach because the balanced feature grid guarantees a nearly uniform ray-grid hitting rate [S1].

## Background
The optimization loops described here originate from the seminal NeRF paper [S2, S6] and subsequent works like EgoNeRF [S1]. The standard NeRF training loop uses Adam with a specific learning rate schedule and MSE loss. EgoNeRF [S1] introduced joint optimization of feature grids and environment maps, along with TV loss for real-world data. The selective L2 regularization and staircase decay schedule [S1] were introduced to stabilize training in large-scale or complex scenes. These methods are widely used in computer vision and graphics for novel view synthesis, 3D reconstruction, and inverse rendering.

## Sources
[S1] Balanced Spherical Grid for Egocentric View Synthesis, https://doi.org/10.1109/cvpr52729.2023.01592, fetched 2026-10-29
