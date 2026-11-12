# Optimization and Parallelization

Also called: Distributed Training, Per-Splat Backprop, Mask Gradient Computation

## Overview
This family encompasses the computational strategies required to efficiently train 3D Gaussian Splatting (3DGS) models at scale. It addresses the high memory bandwidth and compute demands of differentiable rasterization by optimizing the backward pass, distributing parameters across multiple GPUs, and refining gradient computation for auxiliary parameters like masks. It differs from standard differentiable rendering by explicitly managing the dynamic population of Gaussians (densification/pruning) during optimization [5325-training-pipeline.md] and by utilizing per-splat or distributed parallelization granularities to mitigate atomic contention [5989-parallelization-for-backpropagation.md, 7349-grendel-gs.md].

## Shared principles
The training process relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters (e.g., masks).

The forward pass computes the pixel color $\mathbf{c}(\mathbf{x})$ for each pixel $\mathbf{x}$ by alpha-blending Gaussians sorted by depth. The backward pass computes gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all parameters $\theta$.

Key shared mathematical components include:
1.  **Alpha Blending**: The color contribution of a Gaussian $i$ is modulated by its opacity $\alpha_i$ and transmittance $T_i$ from preceding Gaussians.
2.  **Gradient Propagation**: Gradients flow from the loss through the pixel color, through the alpha-blending chain, to the individual Gaussian attributes.
3.  **Parallelization Granularity**: Computation is parallelized across pixels, splats (Gaussians), or batches of views. The standard approach is per-pixel backpropagation, but alternatives exist [5989-parallelization-for-backpropagation.md].
4.  **Memory Management**: Large scenes require managing the Gaussian list in memory, potentially across multiple GPUs or via out-of-core techniques [7349-grendel-gs.md].

## Subtopics

### [MaskGaussian Gradient Computation](4254-maskgaussian-gradient-computation.md)
Computes gradients for per-Gaussian probabilistic masks $\mathcal{M}_i$ that modulate opacity. It explicitly accounts for the interaction between mask-modulated Gaussian contributions and background color transmittance, enabling adaptive pruning [4254-maskgaussian-gradient-computation.md].

### [Training Pipeline](5325-training-pipeline.md)
The iterative loop managing view sampling, rasterization, loss computation, and population management (densification/pruning). It solves the inverse problem of determining Gaussian parameters that minimize rendering error while dynamically adjusting the primitive count [5325-training-pipeline.md].

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
Shifts gradient accumulation granularity from per-pixel to per-splat. This reduces atomic contention and false sharing inherent in standard per-pixel backpropagation, accelerating training while maintaining numerical equivalence [5989-parallelization-for-backpropagation.md].

### [Grendel-GS](7349-grendel-gs.md)
A distributed framework partitioning Gaussians across multiple GPUs. It overcomes single-device memory limits by allowing each GPU to render its local subset and combining results via inter-GPU communication at partition boundaries [7349-grendel-gs.md].

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Provides concrete hyperparameters and optimizations (e.g., SH degree reduction, scene extent correction, half-precision stability) to ensure training stability and speed [9128-3d-gaussian-splatting-implementation-details.md].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [9128-3d-gaussian-splatting-implementation-details.md]. The initial implementation used a standard per-pixel parallelization strategy. Subsequent research identified bottlenecks in memory bandwidth and atomic operations, leading to per-splat backpropagation [5989-parallelization-for-backpropagation.md] and distributed training frameworks like Grendel-GS [7349-grendel-gs.md] to handle scenes with billions of Gaussians. Variants such as MaskGaussian [4254-maskgaussian-gradient-computation.md] and FaCT-GS [5325-training-pipeline.md] introduced auxiliary losses and parameters, requiring specialized gradient computations.

## Sources
