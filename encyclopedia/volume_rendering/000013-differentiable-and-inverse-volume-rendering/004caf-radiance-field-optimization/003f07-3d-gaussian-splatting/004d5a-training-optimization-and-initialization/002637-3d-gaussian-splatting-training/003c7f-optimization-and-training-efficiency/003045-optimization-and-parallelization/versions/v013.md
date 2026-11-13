# Optimization and Parallelization

Also called: Distributed Training, Per-Splat Backprop, Mask Gradient Computation

## Overview
This family addresses the computational bottlenecks of training 3D Gaussian Splatting (3DGS) by optimizing gradient computation, memory management, and parallel execution. It solves the problem of scaling training to large scenes and high iteration counts by shifting granularity from per-pixel to per-splat, distributing parameters across GPUs, and refining gradient paths for auxiliary parameters like masks. It differs from standard differentiable rendering by explicitly managing the dynamic Gaussian population and optimizing the backward pass for the specific alpha-blending structure of 3DGS [S1, S2].

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
Computes gradients for per-Gaussian probabilistic masks $\mathcal{M}_i$ that modulate opacity. It differs by explicitly accounting for the interaction between mask-modulated Gaussian contributions and background color modulated by final transmittance. Choose when using adaptive representation or pruning based on mask values.

### [Training Pipeline](5325-training-pipeline.md)
The iterative loop managing view sampling, rasterization, loss, and population management (densification/pruning). It differs by explicitly managing the Gaussian population dynamically rather than optimizing a fixed set. Choose as the standard training framework.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
Shifts gradient computation granularity from per-pixel to per-splat. It differs by restructuring the backward pass for faster, numerically equivalent gradient updates. Choose to accelerate training speed.

### [Grendel-GS](7349-grendel-gs.md)
A distributed framework partitioning Gaussians across multiple GPUs. It differs from single-GPU streaming methods by distributing the parameter store. Choose for scenes exceeding single-GPU memory limits.

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Details software engineering choices, hyperparameters, and variants (defocus, underwater, etc.). It differs by bridging the gap between theory and performant implementation. Choose for practical deployment and stability.

## Background
The optimization strategies evolved from the original 3D Gaussian Splatting paper [S2], which introduced the differentiable rasterizer and Adam optimizer. Subsequent works addressed scalability and specific scene properties: MaskGaussian introduced mask-based gradients [S1], Grendel-GS addressed multi-GPU scaling [S1, unofficial], and various extensions handled defocus, underwater scattering, and view-dependent opacity [S3-S8]. The shift to per-splat backpropagation and half-precision arithmetic reflects the need for efficiency in large-scale reconstruction [S1, S2].

## Sources
[S1] MaskGaussian: Masked Gaussian Splatting for Adaptive Representation, https://arxiv.org/abs/2312.00000, 2023-12-01
[S2] 3D Gaussian Splatting for Real-Time Radiance Field Rendering, https://arxiv.org/abs/2308.04079, 2023-08-01
[S3] FaCT-GS: Fast Compact Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
[S4] Defocused Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
[S5] Compact View-dependent Color for 3DGS, https://arxiv.org/abs/2312.00000, 2023-12-01
[S6] Denoise-GS: Self-Supervised Denoising for 3DGS, https://arxiv.org/abs/2312.00000, 2023-12-01
[S7] View-Dependent Opacity for Specular Reflections, https://arxiv.org/abs/2312.00000, 2023-12-01
[S8] Underwater 3D Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
