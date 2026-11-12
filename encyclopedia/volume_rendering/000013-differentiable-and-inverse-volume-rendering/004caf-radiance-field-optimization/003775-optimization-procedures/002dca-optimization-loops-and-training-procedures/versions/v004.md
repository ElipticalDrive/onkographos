# Optimization Loops and Training Procedures

Also called: Inverse Rendering Loop

## Overview
This family defines the iterative optimization procedure that minimizes a photometric loss between rendered images and ground-truth observations by updating scene parameters via gradient descent [S1, unofficial]. It differs from standard volume rendering by treating the scene representation as a differentiable function and using stochastic gradient descent to adjust parameters such that the volume rendering equation produces accurate outputs [S5, unofficial][S14, unofficial]. While the loop mechanism is simple, selecting a faithful model, identifiable parameters, a stable estimator, and a good initialization is critical for convergence [S1, unofficial].

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
The standard iterative optimization for neural radiance fields using Adam, MSE loss, and positional encoding, typically running for 1,000,000 steps with a batch size of 1024–2048 rays.

### [Optimization Configuration (Adam + Exponential Decay + Selective L2)](2133-optimization-configuration-adam-exponential-decay-selective-.md)
A specific hyperparameter setup using staircase exponential decay and selective L2 weight decay applied only to MLPs, excluding positional encodings to prevent divergence.

### [Neural Volume Rendering Training (MSE)](3429-neural-volume-rendering-training-mse.md)
The foundational baseline minimizing pixel-wise MSE without geometric priors or complex scheduling, serving as the reference for implicit neural field training.

### [Stage-based Differentiable Volume/Radiance Field Training](4241-stage-based-differentiable-volume-radiance-field-training.md)
A progressive training method alternating between image-based and ray-based sampling, applying stochastic preconditioning with exponential alpha decay, and upsampling parameters between stages.

### [Photometric Optimization](5790-photometric-optimization.md)
An unconstrained optimization task updating latent parameters (shape, pose, appearance) to minimize photometric difference, often jointly optimizing complex latent shape representations and appearance parameters.

### [Staggered Optimization Scheme](5832-staggered-optimization-scheme.md)
A decoupled scheme for 3D fluorescence microscopy that alternates between shape optimization (using narrow-band Fourier Transform) and PSF optimization (using exact FT) to manage computational cost and bias.

### [Optimization Baselines](6675-optimization-baselines.md)
Black-box strategies for non-differentiable renderers that use population-based search distributions (Gaussian) to adapt to the local loss landscape without explicit gradients.

### [Generalized Rendering Models](6681-generalized-rendering-models.md)
Deep neural architectures (e.g., DVRNet, RenderNet) trained end-to-end to optimize volume rendering parameters, learning complex visual feature mappings directly from image-space data.

### [Two-Stage Inverse Rendering Optimization](7387-two-stage-inverse-rendering-optimization.md)
A pipeline decoupling geometry recovery (using approximate differentiable rasterization) from material reflectance optimization (using physically-based ray tracing) to reconstruct triangle meshes and PBR materials.

## Background
The concept of using a differentiable renderer inside an optimizer to estimate pose, geometry, materials, or illumination from images was formalized in the context of differentiable ray tracing [S1, unofficial]. The loop is conceptually simple: initialize parameters, trace rays, compute loss, backpropagate, and update [S1, unofficial]. However, the difficulty lies in choosing a faithful model, identifiable parameters, a stable estimator, and a good initialization [S1, unofficial]. The field has evolved from basic neural volume rendering with MSE loss to complex stage-based, staggered, and two-stage inverse rendering pipelines that handle specific challenges like specular highlights, view-dependent shading, and non-differentiable forward passes.

## Sources
[S1] Differentiable ray tracing — from paths to gradients | Bai Liping, https://bailiping.com/differentiable-ray-tracing/, fetched 2026-10-28 (unofficial)
