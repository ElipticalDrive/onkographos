# Radiance-field optimization

Also called: Differentiable radiance field training, Inverse radiance field optimization

## Overview
Radiance-field optimization is the family of methods that reconstruct 3D scene representations (radiance fields) from 2D observations by minimizing the photometric error between rendered images and ground-truth inputs. It solves the inverse problem of determining scene parameters (density, color, geometry, or camera poses) such that the differentiable volume rendering equation produces images matching the input data. It sets itself apart from explicit forward rendering methods by treating the scene representation as a set of trainable parameters optimized via gradient descent, and from tomographic reconstruction by relying on the differentiability of the rendering operator rather than algebraic inversion of projection data.

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

### [Color Volume Optimization and Density Initialization](842-color-volume-optimization-and-density-initialization.md)
Solves local minima in differentiable volume rendering with non-monotonic transfer functions by first reconstructing a pre-shaded color volume to initialize the density field.

### [Neural Volumes](912-neural-volumes.md)
Synthesizes novel views of dynamic scenes using a deep 3D convolutional network to predict a discretized RGB$\alpha$ voxel grid and a 3D warp field, combining a VAE for latent encoding with differentiable ray-marching.

### [Best Viewpoint Selection via Opacity Entropy](1121-best-viewpoint-selection-via-opacity-entropy.md)
Optimizes camera parameters to maximize visual information by maximizing the entropy of the rendered alpha channel, treating camera pose as a differentiable variable.

### [Regularization losses](1413-regularization-losses.md)
Penalty terms added to the total optimization objective to constrain learned parameters, prevent overfitting, and enforce physical or statistical priors.

### [Hierarchical Grid Optimization](1414-hierarchical-grid-optimization.md)
Training strategies for 3D radiance fields using discrete voxel grids that iteratively refine the grid structure through pruning and subdivision to efficiently optimize high-resolution representations.

### [Volumetric Inverse Rendering with Differentiable Path Tracing](1523-volumetric-inverse-rendering-with-differentiable-path-tracin.md)
Optimizes a 3D volume density grid to match synthetic reference images using path replay backpropagation to compute derivatives of delta tracking and volumetric multiple scattering.

### [3D Gaussian Splatting](1795-3d-gaussian-splatting.md)
Models a radiance field as a collection of learnable 3D Gaussian primitives, optimizing their parameters via differentiable volume rendering to synthesize novel views with real-time speeds.

### [Radiometric Self-Calibration](1820-radiometric-self-calibration.md)
Optimizes per-camera and per-channel radiometric parameters (gain and bias) during reconstruction to correct for photometric errors and intensity mismatches between cameras.

### [TensoRF](3026-tensorf.md)
Models a radiance field using factorized tensors (CP or VM decomposition) for a regular 3D grid, replacing neural networks to achieve faster reconstruction and smaller model sizes.

### [3D Gaussian Splatting Training](3761-3d-gaussian-splatting-training.md)
Non-deep-learning optimization process adjusting explicit 3D Gaussian parameters (centers, covariances, opacities, colors) using differentiable rasterization and adaptive density control.

### [Refractive Index Tomography using Neural Fields](4146-refractive-index-tomography-using-neural-fields.md)
Recovers a 3D refractive index field and luminance field from single-view refracted images by optimizing neural fields that represent these quantities, modeling light propagation through media with spatially varying refractive index.

### [Neural Transient Fields (NeTF)](4760-neural-transient-fields-netf.md)
Models the plenoptic transient field using an MLP to predict transient arrival times from sparse SPAD measurements for Non-Line-of-Sight imaging.

### [Neural Radiance Fields](4972-neural-radiance-fields.md)
Synthesizes novel views by learning a continuous function (typically an MLP) that maps spatial coordinates and viewing directions to volume density and emitted color, representing the scene implicitly.

### [Optimization Loops and Training Procedures](5622-optimization-loops-and-training-procedures.md)
Defines iterative procedures to minimize photometric error by updating parameters, focusing on ray sampling, gradient propagation, and hyperparameter scheduling.

### [Plenoxels](5942-plenoxels.md)
Represents radiance fields using a sparse 3D grid of trilinearly interpolated values with Spherical Harmonic coefficients, replacing MLPs to allow for fast optimization and rendering.

### [HybridNeRF](6363-hybridnerf.md)
Uses a hybrid surface–volume representation that renders most of the scene as surfaces while modeling challenging regions volumetrically using spatially-varying surfaceness parameters.

### [Ref-DVGO](6572-ref-dvgo.md)
Combines the fast hybrid implicit-explicit representation of DVGO with reflection direction reparameterization to achieve high-quality photorealistic novel view synthesis of reflective scenes.

### Local Light Field Fusion
Designed for producing photorealistic novel views for well-sampled forward-facing scenes. It uses a trained 3D convolutional network to directly predict a discretized frustum-sampled RGB$\alpha$ grid (multiplane image or MPI) for each input view, then renders novel views by alpha compositing and blending nearby MPIs into the novel viewpoint. It does not exceed 64 pixels of disparity between input views. It can process a small input dataset in under 10 minutes but produces enormous storage requirements (over 15GB for one "Realistic Synthetic" scene). It frequently fails to estimate correct geometry in datasets with up to 400-500 pixels of disparity between views and blends between different scene representations for rendering different views, resulting in perceptually-distracting inconsistency [S5].

### Inverse Rendering with Adjoint Derivatives
An optimization method that estimates scene optical properties (position, scale, rotation, cross-section, spherical harmonic coefficients) from images using gradient-based optimization with adjoint derivatives of the forward pass. It starts optimization with a small number of primitives and gradually generates new primitives to add detail. It uses the Bound Adam optimizer (an improved Adam optimizer that respects parameter boundaries) and a loss function consisting of L1 error, D-SSIM, plus regularization terms for anisotropy and density of small primitives [S6, unofficial].

### EndoNeRF
A method for deformable surgical scene reconstruction using a canonical radiance field and a time-dependent displacement field, optimized with stereo depth-cueing and supervision. Radiance and density at point $\mathbf{x}$ at time $\tau$ are obtained by querying the canonical field $F_\theta$ at the displaced location: $F_\theta(\mathbf{x} + G_\phi(\mathbf{x}, \tau), \mathbf{d})$. It uses an MLP architecture with 8 fully connected ReLU layers, each with 256 channels. It optimizes an entire spatial-temporal field, resulting in significant time and resource consumption [S7].

### Light Field Representation
A technique to generate high-quality light field representation and integrate it into the GPU-based raycasting pipeline to provide novice audiences access to interactive volume renditions [S8].

### NeuRAD
A neural rendering method for Autonomous Driving (AD) with sensor models (camera + LiDAR) [S9, unofficial].

## Background
Radiance-field optimization emerged as a dominant paradigm in computer graphics and vision for novel view synthesis and inverse rendering. The foundational work on Neural Radiance Fields (NeRF) by Mildenhall et al. (2021) demonstrated the efficacy of learning continuous volumetric representations using MLPs and differentiable volume rendering [S2][S3, S4, unofficial]. Prior to this, methods like Neural Volumes (Mildenhall et al., 2018) explored explicit voxel grids with learned warp fields for dynamic scenes [S1, unofficial][S2]. The field has since expanded to include explicit representations like 3D Gaussian Splatting (Kerbl et al., 2023) and Plenoxels, as well as specialized applications in medical imaging (EndoNeRF) and non-line-of-sight imaging (NeTF) [S2][S6, unofficial][S7]. The core insight is that the volume rendering equation is differentiable with respect to scene parameters, allowing gradient-based optimization to reconstruct 3D scenes from 2D images [S2].

## Sources
[S1] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
[S2] 14 Rendering Volume and Subsurface Scattering - Foundations of Visual ..., https://learnvisualcomputing.github.io/rendering-rte.html, fetched 2026-10-27
[S3] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S4] https://github.com/WangFeng18/3d-gaussian-splatting, https://github.com/WangFeng18/3d-gaussian-splatting, fetched 2026-10-30 (unofficial)
[S5] NeRF: Representing scenes as neural radiance fields for view synthesis, http://www.matthewtancik.com/nerf, fetched 2026-10-27
[S6] Don’t Splat your Gaussians: Volumetric Ray-Traced Primitives for Modeling and Rendering Scattering and Emissive Media (2025), https://qiita.com/matumu20/items/67b33681dc97ccbcc849, fetched 2026-10-27 (unofficial)
[S7] Efficient Deformable Tissue Reconstruction via Orthogonal Neural Plane, https://doi.org/10.1109/tmi.2024.3388559, fetched 2026-10-26
[S8] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
[S9] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
