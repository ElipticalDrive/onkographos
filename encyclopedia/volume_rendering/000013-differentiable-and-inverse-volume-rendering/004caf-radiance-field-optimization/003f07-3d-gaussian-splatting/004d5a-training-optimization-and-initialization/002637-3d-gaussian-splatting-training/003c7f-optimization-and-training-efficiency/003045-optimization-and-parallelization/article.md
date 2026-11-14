# Optimization and Parallelization

Also called: Distributed Training, Parallel Backpropagation, Kernel Optimization

## Overview
This family addresses the computational bottlenecks of training 3D Gaussian Splatting (3DGS), specifically the high memory footprint of large Gaussian populations and the latency of the differentiable rasterization backward pass. It solves the problem of scaling training to large-scale scenes and complex geometries by distributing parameters across multiple GPUs, restructuring gradient computation for hardware efficiency, and optimizing low-level kernel execution. It differs from standard differentiable rendering pipelines by explicitly managing the dynamic population of Gaussians (densification/pruning) in parallel with the optimization loop, and by utilizing specialized CUDA kernels for alpha blending rather than generic automatic differentiation [S2, S3].

## Shared principles
The training process relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters.

Key shared mathematical components include:
1.  **Alpha Blending**: The color contribution of a Gaussian $i$ is modulated by its opacity $\alpha_i$ and transmittance $T_i$ from preceding Gaussians.
2.  **Gradient Propagation**: Gradients flow from the loss through the pixel color, through the alpha-blending chain, to the individual Gaussian attributes.
3.  **Parallelization Granularity**: Computation is parallelized across pixels, splats (Gaussians), or batches of views. The standard approach is per-pixel backpropagation, but alternatives exist [S2].
4.  **Memory Management**: Large scenes require managing the Gaussian list in memory, potentially across multiple GPUs or via out-of-core techniques.

## Subtopics

### [MaskGaussian Gradient Computation](4254-maskgaussian-gradient-computation.md)
Computes gradients for probabilistic masks that modulate Gaussian opacity, enabling adaptive representation and pruning. It differs by explicitly accounting for the interaction between mask-modulated Gaussian contributions and background color transmittance.

### [Training Pipeline](5325-training-pipeline.md)
The iterative optimization loop that reconstructs a 3D scene by managing the Gaussian population through densification, pruning, and splitting. It differs by explicitly modifying the primitive set during optimization rather than optimizing a fixed set.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
A strategy that shifts gradient computation granularity from per-pixel to per-splat (per-Gaussian) to improve training speed. It differs by restructuring the backward pass for faster, numerically equivalent solutions.

### [Grendel-GS](7349-grendel-gs.md)
A distributed optimization framework that partitions Gaussian primitives across multiple GPUs to overcome single-device memory limits. It differs from single-GPU streaming methods by distributing the parameter store across a cluster.

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Details low-level kernel logic, hardware-aware optimizations (half-precision), and specific handling for complex phenomena. It differs by focusing on the gap between theoretical rasterization and practical GPU execution.

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S2]. The initial implementation used a standard PyTorch-based differentiable rasterizer. Subsequent work identified memory and speed bottlenecks, leading to the development of distributed frameworks like Grendel-GS and optimized kernels in libraries such as `gsplat` [S2]. The training pipeline's adaptive control (densification/pruning) was a key innovation to handle varying scene densities [S3].

## Sources
[S1] MaskGaussian Gradient Computation notes
[S2] 3D Gaussian Splatting Implementation Details notes
[S3] Training Pipeline notes
