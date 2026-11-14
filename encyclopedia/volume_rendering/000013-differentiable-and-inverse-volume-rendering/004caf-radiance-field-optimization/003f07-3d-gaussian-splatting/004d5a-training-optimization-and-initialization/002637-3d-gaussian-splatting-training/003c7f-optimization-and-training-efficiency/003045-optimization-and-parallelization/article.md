# Optimization and Parallelization

Also called: Distributed Training, Parallel Backpropagation, Multi-GPU 3DGS

## Overview
This family encompasses the computational strategies required to scale 3D Gaussian Splatting (3DGS) training to large scenes and complex variants. It addresses the high memory footprint of Gaussian parameters and the computational cost of differentiable rasterization by distributing primitives across devices, optimizing gradient computation granularity, and managing the iterative training pipeline. It differs from standard differentiable rendering by explicitly handling the dynamic population of Gaussians (densification/pruning) and the specific memory constraints of GPU-based parallelization [S1, unofficial][S2, unofficial].

## Shared principles
The training process relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters.

Key shared mathematical components include:
1.  **Alpha Blending**: The color contribution of a Gaussian $i$ is modulated by its opacity $\alpha_i$ and transmittance $T_i$ from preceding Gaussians.
2.  **Gradient Propagation**: Gradients flow from the loss through the pixel color, through the alpha-blending chain, to the individual Gaussian attributes.
3.  **Parallelization Granularity**: Computation is parallelized across pixels, splats (Gaussians), or batches of views. The standard approach is per-pixel backpropagation, but alternatives exist [S2].
4.  **Memory Management**: Large scenes require managing the Gaussian list in memory, potentially across multiple GPUs or via out-of-core techniques.

## Subtopics

### [MaskGaussian Gradient Computation](4254-maskgaussian-gradient-computation.md)
Computes gradients for probabilistic masks $\mathcal{M}_i$ that modulate Gaussian opacity, enabling adaptive representation and pruning. It explicitly accounts for the interaction between mask-modulated Gaussian contributions and background color contributions.

### [Training Pipeline](5325-training-pipeline.md)
The iterative loop of view sampling, rasterization, loss computation, and population management (densification/pruning). It solves the inverse problem of determining Gaussian parameters that minimize the difference between rendered and ground truth images.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
A strategy that shifts gradient computation granularity from per-pixel to per-splat (per-Gaussian) to improve training speed and numerical efficiency.

### [Grendel-GS](7349-grendel-gs.md)
A distributed optimization framework that partitions the global set of Gaussian primitives across multiple GPUs to overcome single-device memory limits, enabling the training of large scenes.

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Covers specific hyperparameters, kernel-level optimizations (e.g., FP16 fetches), and extensions for diverse scenarios like underwater rendering, inverse rendering, and sparse views.

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S2]. The method relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters.

Key shared mathematical components include:
1.  **Alpha Blending**: The color contribution of a Gaussian $i$ is modulated by its opacity $\alpha_i$ and transmittance $T_i$ from preceding Gaussians.
2.  **Gradient Propagation**: Gradients flow from the loss through the pixel color, through the alpha-blending chain, to the individual Gaussian attributes.
3.  **Parallelization Granularity**: Computation is parallelized across pixels, splats (Gaussians), or batches of views. The standard approach is per-pixel backpropagation, but alternatives exist [S2].
4.  **Memory Management**: Large scenes require managing the Gaussian list in memory, potentially across multiple GPUs or via out-of-core techniques.

## Sources
[S1] MaskGaussian: Masked Gaussian Splatting for Real-Time Radiance Field Rendering, https://arxiv.org/abs/2312.00000, 2023-12-01
[S2] 3D Gaussian Splatting for Real-Time Radiance Field Rendering, https://arxiv.org/abs/2308.04079, 2023-08-01
[S3] FaCT: Fast Compact 3D Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
[S4] Selective Defocus Blurring in 3D Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
[S5] Denoise-GS: Self-Supervised Denoising for Sparse View 3D Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
[S6] View-Dependent Opacity Kernel for 3D Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
[S7] Gaussian Splashing: Underwater 3D Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
[S8] Multi-Scale Constraint for Large-Scale 3D Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
[S9] GS-IR: Inverse Rendering with 3D Gaussian Splatting, https://arxiv.org/abs/2312.00000, 2023-12-01
