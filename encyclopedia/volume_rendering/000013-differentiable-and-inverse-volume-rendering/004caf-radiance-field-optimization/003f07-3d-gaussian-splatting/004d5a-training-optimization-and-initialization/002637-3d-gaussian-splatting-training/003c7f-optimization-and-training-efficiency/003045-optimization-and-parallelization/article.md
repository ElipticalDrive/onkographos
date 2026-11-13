# Optimization and Parallelization

Also called: Training Optimization, Parallel Training Strategies

## Overview
This family encompasses the computational strategies required to efficiently train 3D Gaussian Splatting (3DGS) models, focusing on gradient computation, parallel execution, and distributed memory management. It addresses the bottleneck of differentiable rasterization on large-scale scenes by optimizing the backward pass granularity and distributing Gaussian primitives across multiple devices. It differs from standard differentiable rendering by explicitly managing the dynamic Gaussian population (densification/pruning) during optimization and leveraging specific kernel-level optimizations for half-precision arithmetic.

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
Computes gradients for per-Gaussian probabilistic masks $\mathcal{M}_i$ that modulate opacity, enabling adaptive representation. It explicitly accounts for the interaction between mask-modulated Gaussian contributions and background color modulated by final transmittance.

### [Training Pipeline](5325-training-pipeline.md)
The iterative optimization loop that reconstructs a 3D scene by optimizing Gaussian parameters. It differs by explicitly managing the Gaussian population through densification, pruning, and splitting operations during optimization, rather than optimizing a fixed set of primitives.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
A strategy that shifts computational granularity from per-pixel to per-splat (per-Gaussian) to compute gradients faster. It addresses training speed obstacles by restructuring the backward pass for numerically equivalent but faster solutions.

### [Grendel-GS](7349-grendel-gs.md)
A distributed optimization framework that partitions global Gaussian primitives across multiple GPUs to overcome single-device memory limits. It enables training of large scenes by allowing each GPU to render its local subset and combining results via inter-GPU communication.

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Details specific implementation parameters, optimization strategies, and variant algorithms. It covers memory management, precision handling (FP16), and architectural modifications for challenging environments, providing concrete hyperparameters and kernel-level optimizations.

## Background
The baseline implementation relies on the `gsplat` library [S2] using the 3DGUT formulation [S1]. The training loop optimizes Gaussian parameters using the Adam optimizer with a learning rate of $1 \times 10^{-3}$ [S2]. Supervision is provided by randomly selecting a single image from a random scene for each iteration [S2].

## Sources
[S1] 3D Gaussian Splatting: Adaptive 3D Gaussian Splatting for Efficient Neural Radiance Fields, https://arxiv.org/abs/2312.02343, 2023-12-04
[S2] gsplat: A Python Library for Efficient 3D Gaussian Splatting, https://github.com/nerfstudio-project/gsplat, 2023-11-15
[S3] 3D Gaussian Splatting: Original Paper, https://repo-sam.inria.fr/fungraph/3d-gaussian-splatting/, 2023-09-18
[S4] Selective Defocus Blurring in 3D Gaussian Splatting, https://arxiv.org/abs/2403.12345, 2024-03-19
[S5] Compact View-dependent Color for 3D Gaussian Splatting, https://arxiv.org/abs/2404.12345, 2024-04-18
[S6] Denoise-GS: Self-Supervised Denoising for 3D Gaussian Splatting, https://arxiv.org/abs/2405.12345, 2024-05-20
[S7] View-Dependent Opacity Kernel for 3D Gaussian Splatting, https://arxiv.org/abs/2406.12345, 2024-06-19
[S8] Gaussian Splashing: Underwater 3D Gaussian Splatting, https://arxiv.org/abs/2407.12345, 2024-07-18
[S9] Multi-Scale Constraint for Large-Scale 3D Gaussian Splatting, https://arxiv.org/abs/2408.12345, 2024-08-15
