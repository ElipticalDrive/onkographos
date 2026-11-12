# Optimization and Parallelization


## Overview
This family addresses the computational bottlenecks of 3D Gaussian Splatting (3DGS) training, specifically the high memory bandwidth requirements of the backward pass and the memory limits of single-GPU training. It differs from standard differentiable rendering optimization by focusing on the unique data parallelism and memory management challenges posed by the large number of primitives (billions of Gaussians) and the per-pixel alpha-blending dependency chain. Key strategies include shifting backpropagation granularity from per-pixel to per-splat to reduce atomic contention [5989-parallelization-for-backpropagation.md], distributing the parameter store across multiple GPUs [7349-grendel-gs.md], and optimizing implementation details such as precision and hyperparameters [9128-3d-gaussian-splatting-implementation-details.md].

## Shared principles
The training process relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters (e.g., masks).

The forward pass computes the pixel color $\mathbf{c}(\mathbf{x})$ for each pixel $\mathbf{x}$ by alpha-blending Gaussians sorted by depth. The backward pass computes gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all parameters $\theta$.

Key shared mathematical components include:
1.  **Alpha Blending**: The color contribution of a Gaussian $i$ is modulated by its opacity $\alpha_i$ and transmittance $T_i$ from preceding Gaussians.
2.  **Gradient Propagation**: Gradients flow from the loss through the pixel color, through the alpha-blending chain, to the individual Gaussian attributes.
3.  **Parallelization Granularity**: Computation is parallelized across pixels, splats (Gaussians), or batches of views. The standard approach is per-pixel backpropagation, but alternatives exist [5989-parallelization-for-backpropagation.md].
4.  **Memory Management**: Large scenes require managing the Gaussian list in memory, potentially across multiple GPUs or via out-of-core techniques.

## Subtopics

### [MaskGaussian Gradient Computation](4254-maskgaussian-gradient-computation.md)
Computes gradients for probabilistic masks $\mathcal{M}_i$ that modulate Gaussian opacity, enabling adaptive representation. It explicitly accounts for the interaction between mask-modulated Gaussian contributions and background color contributions, which is distinct from standard backpropagation.

### [Training Pipeline](5325-training-pipeline.md)
The iterative optimization loop that reconstructs a 3D scene. It differs by explicitly managing the Gaussian population through densification, pruning, and splitting operations during optimization, rather than optimizing a fixed set of primitives.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
Shifts computational granularity from per-pixel to per-splat (per-Gaussian). It solves the high atomic contention and false sharing inherent in standard per-pixel backpropagation, where thousands of threads simultaneously update the same Gaussian parameters.

### [Grendel-GS](7349-grendel-gs.md)
A distributed optimization framework that partitions the global set of Gaussian primitives across multiple GPUs. It overcomes single-device memory limits by allowing each GPU to render its local subset and combining results via inter-GPU communication, unlike single-GPU streaming approaches.

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Details specific hyperparameters, data structures, and optimization strategies (e.g., half-precision rasterization, corrected scene extent) required to implement the training pipeline efficiently, addressing the gap between theoretical formulation and practical GPU constraints.

## Background
The optimization and parallelization techniques for 3D Gaussian Splatting emerged alongside the original 3DGS paper [S2] and subsequent works addressing its scalability and training efficiency. The standard per-pixel backpropagation approach, while numerically stable, suffers from significant memory bandwidth and atomic operation bottlenecks when dealing with scenes containing millions of Gaussians [5989-parallelization-for-backpropagation.md]. Subsequent research introduced per-splat backpropagation [5989-parallelization-for-backpropagation.md] and multi-GPU distributed training frameworks [7349-grendel-gs.md] to enable training on scenes with billions of primitives. Implementation optimizations, such as using lower SH degrees and half-precision arithmetic, have also been shown to significantly reduce training time without compromising visual quality [9128-3d-gaussian-splatting-implementation-details.md].

## Sources
[S1] 3D Gaussian Splatting: Adaptive Density Control via Gradient-Based Pruning, 2023.
[S2] 3D Gaussian Splatting for Real-Time Radiance Field Rendering, 2023.
[S3] FaCT-GS: Fast Compact Training of 3D Gaussian Splatting, 2024.
[S4] StreetGaussian: Real-Time 3D Gaussian Splatting for Large-Scale Street Scene Navigation, 2024.
[S5] Hard Gaussian Growing: Preventing Over-Large Gaussians in 3D Gaussian Splatting, 2024.
[S6] Optimizing 3D Gaussian Splatting: Practical Tips for Faster Training, 2024.
