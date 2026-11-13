# Optimization and Parallelization

Also called: Distributed Training, Per-Splat Backprop, Mask Gradient Computation

## Overview
This family encompasses the computational strategies required to train 3D Gaussian Splatting (3DGS) efficiently at scale. It addresses the high memory and compute costs of differentiable rasterization by introducing parallelization granularities beyond standard per-pixel processing, distributed memory management across multiple GPUs, and specialized gradient computations for auxiliary parameters like probabilistic masks. It differs from standard differentiable rendering by explicitly managing the dynamic population of Gaussians (densification/pruning) and optimizing specific variants such as view-dependent opacity or underwater scattering [S1, S2, S3].

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
Computes gradients for per-Gaussian probabilistic masks $\mathcal{M}_i$ that modulate opacity. It explicitly accounts for the interaction between mask-modulated Gaussian contributions and background color transmittance. Choose this when using adaptive representation or pruning based on mask parameters.

### [Training Pipeline](5325-training-pipeline.md)
The iterative loop managing view sampling, rasterization, loss computation, and population management (densification/pruning). It differs from static optimization by dynamically modifying the Gaussian set. Choose this as the foundational training loop.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
Shifts gradient computation granularity from per-pixel to per-splat (per-Gaussian) for faster, numerically equivalent updates. Choose this to address training speed bottlenecks in standard backpropagation.

### [Grendel-GS](7349-grendel-gs.md)
A distributed framework partitioning Gaussians across multiple GPUs to overcome single-device memory limits. It combines local rendering with inter-GPU communication at partition boundaries. Choose this for large-scale scenes exceeding single GPU memory.

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Covers specific hyperparameters, half-precision stability, and specialized variants (defocus blur, view-dependent opacity, underwater scattering). Choose this for tuning standard training or implementing specific scene variants.

## Background
The optimization strategies for 3D Gaussian Splatting evolved from the original 3DGS paper [S2] and subsequent works addressing scalability and specific rendering artifacts. The standard pipeline uses Adam optimization with L1+SSIM loss [S2, S3]. Distributed training approaches like Grendel-GS [S1, unofficial] and parallelization techniques [S1] were developed to handle large-scale datasets. Specialized variants address defocus [S4], view-dependent effects [S5, S7], underwater scattering [S8], and sparse-view denoising [S6]. Mask-based optimization [S1, S9] enables adaptive pruning and representation.

## Sources
[S1] MaskGaussian Gradient Computation notes
[S2] 3D Gaussian Splatting Implementation Details notes
[S3] Training Pipeline notes
[S4] Selective Defocus Blurring notes
[S5] Compact View-dependent Color notes
[S6] Denoise-GS notes
[S7] View-Dependent Opacity Kernel notes
[S8] Underwater Rendering notes
[S9] Gaussian Reduction notes
