# Optimization and Parallelization


## Overview
This family encompasses the computational strategies required to efficiently train 3D Gaussian Splatting (3DGS) models, focusing on gradient computation, parallel execution, and distributed memory management. It addresses the bottleneck of differentiable rasterization by optimizing the backward pass and enabling training on scenes larger than a single GPU's memory capacity. It differs from standard differentiable rendering by explicitly managing the dynamic population of Gaussians (densification/pruning) and leveraging per-splat or per-GPU parallelization granularities to accelerate convergence [S1, unofficial][S2, unofficial][S3].

## Shared principles
The training process relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters (e.g., masks).

The forward pass computes the pixel color $\mathbf{c}(\mathbf{x})$ for each pixel $\mathbf{x}$ by alpha-blending Gaussians sorted by depth. The backward pass computes gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all parameters $\theta$.

Key shared mathematical components include:
1.  **Alpha Blending**: The color contribution of a Gaussian $i$ is modulated by its opacity $\alpha_i$ and transmittance $T_i$ from preceding Gaussians.
2.  **Gradient Propagation**: Gradients flow from the loss through the pixel color, through the alpha-blending chain, to the individual Gaussian attributes.
3.  **Parallelization Granularity**: Computation is parallelized across pixels, splats (Gaussians), or batches of views. The standard approach is per-pixel backpropagation, but alternatives exist [S2].
4.  **Memory Management**: Large scenes require managing the Gaussian list in memory, potentially across multiple GPUs or via out-of-core techniques.

## Subtopics

### [MaskGaussian Gradient Computation](4254-maskgaussian-gradient-computation.md)
Computes gradients for per-Gaussian probabilistic masks $\mathcal{M}_i$ that modulate opacity, enabling adaptive representation. It differs by explicitly accounting for the interaction between mask-modulated Gaussian contributions and background color modulated by final transmittance.

### [Training Pipeline](5325-training-pipeline.md)
The iterative loop solving the inverse problem of determining Gaussian parameters. It differs by explicitly managing the Gaussian population through densification, pruning, and splitting during optimization, rather than optimizing a fixed set.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
Shifts computational granularity from per-pixel to per-splat (per-Gaussian) to accelerate gradient computation and attribute updates.

### [Grendel-GS](7349-grendel-gs.md)
A distributed framework partitioning Gaussians across multiple GPUs to overcome single-device memory limits. Unlike [TideGS](7350-tidegs.md), it distributes the parameter store across a cluster rather than streaming from disk.

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Provides concrete software engineering, hyperparameter configurations, and kernel-level details (e.g., FP16 precision, `gsplat` library usage) required to implement the shared principles.

## Background
The optimization strategies for 3D Gaussian Splatting emerged from the original 3DGS paper [S2] and subsequent extensions addressing scalability and accuracy. The standard training pipeline uses Adam optimization with specific learning rates and loss weights [S2][S8]. Distributed training approaches like Grendel-GS were developed to handle large-scale scenes [S1, unofficial]. Implementation details often rely on libraries like `gsplat` and `torchsparse` [S2].

## Sources
[S1] MaskGaussian Gradient Computation notes
[S2] 3D Gaussian Splatting Implementation Details notes
[S3] Training Pipeline notes
[S8] 3D Gaussian Splatting Implementation Details notes
