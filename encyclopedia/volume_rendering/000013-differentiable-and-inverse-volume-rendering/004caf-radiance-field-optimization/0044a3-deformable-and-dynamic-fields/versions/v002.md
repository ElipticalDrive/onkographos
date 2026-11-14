# Deformable and Dynamic Fields

Also called: Dynamic Neural Radiance Fields, Time-dependent Radiance Fields

## Overview
Deformable and Dynamic Fields extend static radiance field optimization to model scenes that change over time or undergo non-rigid deformation [G]. Unlike static methods that assume a fixed geometry, these methods introduce a temporal or deformation component to map input coordinates to a canonical space, enabling the synthesis of novel views for moving or deforming objects [G]. They are distinguished from static voxel methods (e.g., DeepVoxels) and static MLP-based methods (e.g., NeRF) by their explicit handling of motion, either through learned warp fields or displacement mappings [G].

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps [G]. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses) [G].

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [G]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [G].

Differentiation strategies fall into three main categories [G]:
1. **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph [G]. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [G].
2. **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input [G]. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [G].
3. **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [G].

## Subtopics

### [Neural Volumes](912-neural-volumes.md)
Neural Volumes (NV) synthesizes novel views of dynamic scenes by optimizing a deep 3D convolutional network to predict a discretized RGB$\alpha$ voxel grid and a 3D warp field [912-neural-volumes.md]. It solves the problem of rendering semi-transparent, dynamic objects (such as smoke or hair) from sparse multi-view video input without requiring explicit geometry reconstruction or depth sensors [912-neural-volumes.md]. Unlike implicit radiance fields that map coordinates to color via MLPs, NV uses an explicit voxel grid, trading memory for rendering speed and interpretability [912-neural-volumes.md]. It sets itself apart from static voxel methods like DeepVoxels by introducing a learnable warp field to model non-rigid motion and from MLP-based methods like NeRF by avoiding the prohibitive computational cost of evaluating deep networks at every ray-marching step [912-neural-volumes.md].

### EndoNeRF
EndoNeRF is a method for deformable surgical scene reconstruction that utilizes a canonical radiance field and a time-dependent displacement field, optimized with stereo depth-cueing and supervision [5540-endonerf.md]. It addresses the challenge of reconstructing non-rigid tissue dynamics in endoscopic videos by mapping input space-time coordinates to a static canonical space via a learned displacement field [5540-endonerf.md]. Unlike static radiance field methods, it explicitly models temporal deformation, allowing for the querying of radiance and density at any continuous time step [5540-endonerf.md].

### [Fluid Motion Reconstruction Framework](10672-fluid-motion-reconstruction-framework.md)
A computational framework to efficiently reconstruct a temporally coherent sequence of fluid density fields from monocular video, aligning density fields with input images while conforming to physical priors [10672-fluid-motion-reconstruction-framework.md].

### Binary Feature Encoding for Dynamic Scenes
#### definition
- An adaptation of the binary feature encoding technique applied to time-aware voxel features to enable highly compact reconstruction of dynamic scenes without conventional dense feature encoding [S1].
#### step
- 1. Initialize TiNeuVox base model with time-aware voxel features. 2. Replace conventional feature encoding with binary feature encoding on the voxel grid. 3. Train the model on dynamic scene datasets (D-NeRF, HyperNeRF). 4. Evaluate storage size and reconstruction metrics [S1].
#### number
- TiNeuVox-S on D-NeRF (synthetic): Size reduced from 7.63 MB to 0.59 MB; PSNR drops from 31.03 to 28.68; SSIM drops from 0.957 to 0.938 [S1].
- TiNeuVox-B on D-NeRF (synthetic): Size reduced from 47.51 MB to 3.88 MB; PSNR drops from 33.02 to 31.23; SSIM drops from 0.972 to 0.961 [S1].
- TiNeuVox-B on HyperNeRF (real): Size reduced from 47.85 MB to 3.90 MB; PSNR drops from 24.20 to 23.87; MS-SSIM drops from 0.835 to 0.826 [S1].
- Achieves 0.6 MB storage for the synthetic jumpingjacks dynamic scene [S1].

## Background
The need for dynamic scene representation arises from the limitations of static NeRF-like methods in capturing temporal variations [G]. Neural Volumes [912-neural-volumes.md] was among the early works to introduce explicit voxel grids with warp fields for dynamic scenes [G]. EndoNeRF [5540-endonerf.md] later adapted these concepts for the specific challenges of endoscopic surgery, such as non-rigid tissue deformation and limited viewpoint coverage [G].

## Sources
[S1] Binary Radiance Fields, https://doi.org/10.52202/075280-2438, fetched 2026-10-26
