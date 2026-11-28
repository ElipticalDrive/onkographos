# Optimization Loops and Training Procedures

Also called: Inverse Rendering Loop

## Overview
This family defines the iterative optimization procedure that uses a differentiable renderer to estimate scene parameters (geometry, materials, illumination, or camera poses) from observed images or measurements [S1, unofficial]. It solves the inverse rendering problem by minimizing the difference between rendered outputs and ground-truth targets via gradient-based or gradient-free optimization [S1, unofficial]. It sets itself apart from its siblings by establishing the fundamental control flow—sampling, forward rendering, loss computation, and parameter update—that serves as the baseline for all advanced training strategies, ranging from standard NeRF loops to black-box search methods [S8][S14, unofficial].

## Shared principles
All methods in this family rely on the differentiability of the forward rendering operator and the propagation of gradients through the integration steps. The core mechanism is the evaluation of the volume rendering equation and the computation of gradients with respect to scene parameters (density, geometry, transfer functions, camera poses).

The standard emission-absorption model computes the color $\hat{C}(\mathbf{r})$ for a ray $\mathbf{r}$ as [S1, unofficial][S5, unofficial]:

$$
\hat{C}(\mathbf{r}) = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i
$$

where $\sigma_i$ is the density, $\delta_i$ is the step size, $\mathbf{c}_i$ is the color, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the transmittance [S1, unofficial][S5, unofficial].

Differentiation strategies fall into three main categories:
1.  **Reverse-Mode Automatic Differentiation (AD):** Standard backpropagation through the computational graph. This requires storing $O(kn)$ intermediate states (where $n$ is pixels and $k$ is samples), leading to high memory usage [S5, unofficial].
2.  **Adjoint Method:** Computes gradients by propagating adjoint variables backward through the rendering algorithm, evaluating the chain rule from output to input. This has computational complexity largely constant in the number of parameters, making it preferable for optimizing many parameters like the transfer function [S1, unofficial][S3].
3.  **Analytic Inversion / Explicit Gradients:** Uses a-priori knowledge of the DVR operations to invert the order of operations or derive closed-form gradients, reducing memory consumption to $O(n)$ by avoiding the storage of intermediate results [S5, unofficial].

## Subtopics

### [NeRF Training Loop](970-nerf-training-loop.md)
The NeRF Training Loop is the iterative optimization procedure that learns a continuous volumetric scene representation by minimizing the difference between rendered images and ground truth observations. It solves the inverse rendering problem by updating the parameters of a coordinate-based neural network (MLP) via gradient descent. It sets itself apart from its siblings by establishing the fundamental stochastic optimization cycle—ray sampling, differentiable volume rendering, and loss minimization—that serves as the baseline for all advanced NeRF variants.

### [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md)
This configuration defines the hyperparameter setup and regularization strategy used to optimize neural radiance fields and volume rendering networks. It combines the Adam optimizer with a staircase exponential learning rate decay schedule and selective L2 weight decay applied only to specific network components (MLPs) while excluding positional encodings and appearance embeddings. This setup differs from standard NeRF training loops by explicitly masking L2 regularization to prevent divergence in embedding spaces and by using a decay schedule that holds the initial learning rate for 10,000 steps before decaying by a factor of 1/3 every 10,000 steps.

### [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md)
Neural Volume Rendering Training (MSE) is the foundational optimization procedure for training implicit neural radiance fields by minimizing the mean squared error between rendered and ground-truth images. It solves the inverse problem of recovering a continuous volumetric scene representation from discrete 2D observations. It sets itself apart from sibling methods by relying exclusively on pixel-wise photometric fidelity without additional geometric priors (like Eikonal regularization), semantic constraints (like handpainted inversion), or complex multi-stage scheduling, serving as the baseline differentiable rendering loop.

### [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md)
A stage-based optimization loop for differentiable volume or radiance field reconstruction that alternates between image-based and ray-based sampling modes, applies stochastic preconditioning with exponential alpha decay, and optionally upsamples parameters between stages. It differs from standard NeRF training loops by explicitly managing memory via ray batching and progressive resolution upsampling, while using a specific stochastic preconditioning schedule to stabilize gradient updates.

### [Photometric Optimization](5790-photometric-optimization.md)
Photometric optimization is a training procedure that minimizes the difference between rendered images and observed target images by updating scene parameters via gradient descent. It sets itself apart from its siblings by explicitly formulating the inverse problem as an unconstrained optimization task where latent parameters (such as shape vectors, pose angles, and appearance) are updated to minimize the photometric difference between a rendered image and an observed target. Unlike [NeRF Training Loop](970-nerf-training-loop.md), which may focus on standard coordinate-based MLPs, this approach often jointly optimizes complex latent shape representations and appearance parameters.

### [Staggered Optimization Scheme](5832-staggered-optimization-scheme.md)
The Staggered Optimization Scheme is a training procedure for differentiable volume rendering that decouples the joint optimization of geometric shape parameters and Point Spread Function (PSF) parameters into two sequential sub-problems. It solves the problem of optimization instability and computational bias caused by the strong coupling between shape and PSF, where optimizing them simultaneously can lead to degenerate shapes or incorrect blur estimation. Unlike joint optimization methods that update all parameters in a single step, this scheme alternates between a shape optimization phase and a PSF optimization phase, allowing each to converge more stably. It is specifically designed for differentiable mesh-based rendering in fluorescence microscopy, where the forward pass is significantly slower than the backward pass.

### [Optimization Baselines](6675-optimization-baselines.md)
This family provides black-box optimization strategies for inverse rendering problems where the forward rendering operator $f$ is non-differentiable or gradient computation is prohibitively expensive. Unlike gradient-based siblings that rely on automatic differentiation or adjoint methods, these methods treat the renderer as an opaque function, minimizing a photometric loss $\mathcal{L}$ by iteratively sampling and evaluating parameter candidates. The family is distinguished by its reliance on population-based search distributions (typically Gaussian) that adapt to the local loss landscape without explicit gradient information, offering robustness against local minima and Monte Carlo noise.

### [Generalized Rendering Models](6681-generalized-rendering-models.md)
Generalized Rendering Models are deep neural architectures trained to optimize volume rendering parameters end-to-end, enabling consistent visual output across subjects with high inter-patient variations in clinical data. Unlike manual transfer functions or simple lookup tables, these models learn complex visual feature mappings (such as colorization and view-dependent shading) directly from image-space data, bypassing the need for explicit intensity-based feature engineering.

### [Two-Stage Inverse Rendering Optimization](7387-two-stage-inverse-rendering-optimization.md)
Two-Stage Inverse Rendering Optimization is a pipeline that decouples the recovery of scene geometry from material reflectance by utilizing two distinct differentiable rendering backends. It solves the ill-posed nature of inverse rendering by first optimizing a triangle mesh using a fast, approximate differentiable renderer focused on silhouette consistency, and subsequently optimizing diffuse and specular albedo textures using a physically-based ray-tracing renderer. It differs from NeRF, IDR, and PhySG by explicitly reconstructing triangle meshes and Cook-Torrance PBR materials rather than implicit SDFs or Spherical Gaussians, enabling direct deployment in graphics engines and superior handling of specular highlights.

### 4DGS Baseline Implementation
This variant adapts the standard 3D Gaussian Splatting optimization for multi-view video sequences. It optimizes the first frame for 3,000 iterations, and then optimizes the full multi-view video sequence for 14,000 iterations [S2].

## Background
The concept of using differentiable renderers within an optimizer loop was formalized in the context of differentiable ray tracing and volume rendering. The pseudocode for the inverse problem loop is conceptual: initialize parameters $\theta$, then iterate $K$ times to compute a prediction via tracing, calculate a distance loss against a target, compute the gradient via backward pass, and update $\theta$ using an optimizer [S1, unofficial]. The choice of a faithful model, identifiable parameters, a stable estimator, and a good initialization is difficult, even if the loop is simple [S1, unofficial].

## Sources
[S1] Differentiable ray tracing — from paths to gradients | Bai Liping, https://bailiping.com/differentiable-ray-tracing/, fetched 2026-10-28 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
