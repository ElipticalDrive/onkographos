# Optimization and Parallelization


## Overview
This family encompasses the computational strategies required to efficiently train 3D Gaussian Splatting (3DGS) models, focusing on gradient computation, parallel execution, and distributed memory management. It addresses the bottleneck of differentiable rasterization on large-scale scenes by optimizing the backward pass granularity, distributing Gaussian primitives across multiple GPUs, and refining implementation details for numerical stability. It differs from standard differentiable rendering by explicitly managing the dynamic population of Gaussians (densification/pruning) and leveraging specific hardware-aware optimizations like half-precision fetches and per-splat gradient accumulation.

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
Computes gradients for per-Gaussian probabilistic masks $\mathcal{M}_i$ that modulate opacity. It explicitly accounts for the interaction between mask-modulated Gaussian contributions and background color, enabling adaptive representation and pruning.

### [Training Pipeline](5325-training-pipeline.md)
The iterative loop managing view sampling, rasterization, loss computation, and population management (densification/pruning). It differs by explicitly modifying the Gaussian set during optimization rather than optimizing a fixed set.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
Shifts gradient computation granularity from per-pixel to per-splat (per-Gaussian) to accelerate training. It provides a numerically equivalent but faster formulation for gradient updates.

### [Grendel-GS](7349-grendel-gs.md)
A distributed framework partitioning Gaussians across multiple GPUs to overcome single-device memory limits. It combines local rendering results via inter-GPU communication, unlike single-GPU streaming methods.

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Provides concrete hyperparameters, CUDA-level optimizations (e.g., half-precision fetches), and stability guards. It bridges the gap between theoretical pipelines and practical execution.

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S8]. The optimization techniques described herein evolved to address the computational cost of training on large scenes and the memory constraints of single GPUs. Distributed approaches like Grendel-GS and efficiency improvements like per-splat backpropagation were developed to scale the method to city-scale environments and real-time training speeds.

## Sources
[S1] MaskGaussian Gradient Computation notes
[S2] 3D Gaussian Splatting Implementation Details notes
[S3] Training Pipeline notes
[S4] 3D Gaussian Splatting Implementation Details notes
[S8] 3D Gaussian Splatting Implementation Details notes
[S11] 3D Gaussian Splatting Implementation notes
[S2] Training Pipeline notes (unofficial)
[S3] Training Pipeline notes (unofficial)
[S1] Grendel-GS notes (unofficial)
[S1] 3D Gaussian Splatting Implementation Details notes (unofficial)
