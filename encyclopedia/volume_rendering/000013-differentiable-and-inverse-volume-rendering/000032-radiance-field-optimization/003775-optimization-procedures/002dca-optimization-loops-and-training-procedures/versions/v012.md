# Optimization Loops and Training Procedures

Also called: Inverse Rendering Loop

## Overview
This family defines the iterative optimization procedure that minimizes a photometric loss between rendered images and ground-truth observations by updating scene parameters (geometry, materials, illumination, or radiance field weights) [S1, unofficial]. It solves the inverse problem of recovering 3D scene properties from 2D data by treating the rendering process as a differentiable function [S1, S2, S3]. It sets itself apart from its siblings by providing the generic loop structure (forward render, loss compute, backward pass, update) upon which specific strategies like [NeRF Training Loop](970-nerf-training-loop.md), [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md), and [Photometric Optimization](5790-photometric-optimization.md) are built.

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

### [NeRF Training Loop](970-nerf-training-loop.md)
The NeRF Training Loop is the iterative optimization procedure that minimizes the photometric reconstruction error between rendered images and ground-truth observations by updating the parameters of a neural radiance field. It solves the inverse problem of recovering a continuous 3D scene representation from a set of 2D images by treating the scene as a learnable function $F_\Theta$ mapping 3D coordinates and viewing directions to volume density and color. It sets itself apart from evaluation by using stochastic ray sampling, adaptive perturbations, and chunked processing to manage memory and enable gradient descent, whereas evaluation disables these for deterministic results.

### [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md)
This configuration defines the standard hyperparameter setup for training neural radiance fields and implicit volume representations, combining the Adam optimizer with a staircase exponential learning rate decay and selective L2 weight decay. It addresses the problem of stabilizing gradient descent and preventing divergence in high-dimensional, ill-conditioned optimization landscapes typical of differentiable volume rendering. It sets itself apart from generic optimizers by explicitly disabling regularization on positional encodings and appearance embeddings while applying it to network weights, and by using a delayed decay schedule to allow initial convergence.

### [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md)
Neural Volume Rendering Training (MSE) is the foundational optimization procedure for implicit neural radiance fields, where a neural network is trained to minimize the Mean Squared Error (MSE) between rendered images and ground-truth observations. It solves the problem of learning continuous volumetric or radiance representations from 2D image supervision by treating the rendering equation as a differentiable operator. It sets itself apart from sibling methods like [NeRF Training Loop](970-nerf-training-loop.md) by focusing specifically on the core photometric loss minimization without the specific architectural constraints or hierarchical sampling strategies often associated with the canonical NeRF implementation, and differs from [Photometric Optimization](5790-photometric-optimization.md) by strictly adhering to the L2 loss metric rather than perceptual or structural similarity metrics.

### [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md)
A stage-based optimization loop for differentiable volume or radiance field reconstruction that alternates between image-based and ray-based sampling modes, applies stochastic preconditioning with exponential alpha decay, and optionally upsamples parameters between stages. It differs from standard NeRF training loops by explicitly managing memory via ray batching and progressive resolution upsampling, while using a specific stochastic preconditioning schedule to stabilize gradient updates.

### [Photometric Optimization](5790-photometric-optimization.md)
Photometric Optimization is a training procedure for differentiable volume rendering that minimizes the pixel-wise difference between rendered images and ground-truth observations. It solves the inverse problem of recovering scene geometry, appearance, and lighting parameters by treating the rendering process as a differentiable function. It sets itself apart from [NeRF Training Loop](970-nerf-training-loop.md) by explicitly defining the loss landscape and optimization targets (such as environment maps or per-vertex deformations) rather than relying on a generic training loop structure, and it differs from [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md) by focusing on the *what* (the loss and parameters) rather than the *how* (the optimizer hyperparameters).

### [Staggered Optimization Scheme](5832-staggered-optimization-scheme.md)
The Staggered Optimization Scheme is a training procedure for differentiable volume rendering that decouples the joint optimization of geometric shape parameters and Point Spread Function (PSF) parameters into two sequential sub-problems. It solves the problem of optimization instability and computational bias caused by the strong coupling between shape and PSF, where optimizing them simultaneously can lead to degenerate shapes or incorrect blur estimation. Unlike joint optimization methods that update all parameters in a single step, this scheme alternates between a shape optimization phase and a PSF optimization phase, allowing each to converge more stably. It is specifically designed for differentiable mesh-based rendering in fluorescence microscopy, where the forward pass is significantly slower than the backward pass.

### [Optimization Baselines](6675-optimization-baselines.md)
This family provides black-box optimization strategies for inverse rendering problems where the forward rendering operator $f$ is non-differentiable or gradient computation is prohibitively expensive. Unlike gradient-based siblings that rely on automatic differentiation or adjoint methods, these methods treat the renderer as an opaque function, minimizing a photometric loss $\mathcal{L}$ by iteratively sampling and evaluating parameter candidates. The family is distinguished by its reliance on population-based search distributions (typically Gaussian) that adapt to the local loss landscape without explicit gradient information, offering robustness against local minima and Monte Carlo noise.

### [Generalized Rendering Models](6681-generalized-rendering-models.md)
Generalized Rendering Models are deep neural architectures trained to optimize volume rendering parameters end-to-end, enabling consistent visual output across subjects with high inter-patient variations in clinical data. Unlike manual transfer functions or simple lookup tables, these models learn complex visual feature mappings (such as colorization and view-dependent shading) directly from image-space data, bypassing the need for explicit intensity-based feature engineering.

### [Two-Stage Inverse Rendering Optimization](7387-two-stage-inverse-rendering-optimization.md)
Two-Stage Inverse Rendering Optimization is a pipeline that decouples the recovery of scene geometry from material reflectance by utilizing two distinct differentiable rendering backends. It solves the ill-posed nature of inverse rendering by first optimizing a triangle mesh using a fast, approximate differentiable renderer focused on silhouette consistency, and subsequently optimizing diffuse and specular albedo textures using a physically-based ray-tracing renderer. It differs from NeRF, IDR, and PhySG by explicitly reconstructing triangle meshes and Cook-Torrance PBR materials rather than implicit SDFs or Spherical Gaussians, enabling direct deployment in graphics engines and superior handling of specular highlights.

### [DressiAD Optimization Loop](10728-dressiad-optimization-loop.md)
The DressiAD Optimization Loop is a define-and-run differentiable rendering optimization procedure that constructs a computational graph, marks variables for gradient tracking, and executes forward/backward passes using Stochastic Gradient Descent (SGD). It distinguishes itself from standard AD libraries by packing the backward pass into Vulkan command buffers, allowing the optimization loop to execute directly on the GPU without CPU-side backpropagation overhead.

### 4DGS Baseline Implementation
Optimizes the first frame for 3,000 iterations, then optimizes the full multi-view video sequence for 14,000 iterations [S2].

## Background
The concept of using a differentiable renderer inside an optimizer to estimate pose, geometry, materials, or illumination from images is foundational to inverse rendering [S1, unofficial]. The loop structure is simple: initialize parameters, trace rays to predict observations, compute distance to target, compute gradients, and update parameters [S1, unofficial]. However, choosing a faithful model, identifiable parameters, a stable estimator, and a good initialization is difficult, even if the loop is simple [S1, unofficial]. The method originated from the realization that gradients could be propagated through rendering equations, enabling gradient-based optimization of scene parameters [S1, unofficial]. It is used in neural radiance fields (NeRF), differentiable mesh rendering, and inverse rendering of materials and lighting.

## Sources
[S1] Differentiable ray tracing — from paths to gradients | Bai Liping, https://bailiping.com/differentiable-ray-tracing/, fetched 2026-10-28 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
