# Optimization and Parallelization

Also called: Distributed 3DGS Training, Per-Splat Backpropagation, Mask Gradient Computation

## Overview
This family addresses the computational bottlenecks of training 3D Gaussian Splatting (3DGS), specifically the high memory usage, slow convergence, and scalability limits of the standard differentiable rasterization pipeline. It solves the problem of optimizing billions of Gaussian primitives across multiple GPUs and optimizing specific parameters like probabilistic masks. It differs from standard differentiable rendering by introducing specialized gradient computation paths (e.g., for masks), restructuring the backward pass granularity (per-splat vs. per-pixel), and distributing the parameter store across hardware clusters. Key siblings include [MaskGaussian Gradient Computation](4254-maskgaussian-gradient-computation.md), [Training Pipeline](5325-training-pipeline.md), [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md), [Grendel-GS](7349-grendel-gs.md), and [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md).

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
Computes the gradient of a loss function $\mathcal{L}$ with respect to per-Gaussian probabilistic masks $\mathcal{M}_i$. It differs from standard backpropagation by explicitly accounting for the interaction between the mask-modulated Gaussian contribution and the background color contribution, which is modulated by the final transmittance. Choose this when using probabilistic masks for adaptive representation and pruning.

### [Training Pipeline](5325-training-pipeline.md)
The iterative optimization loop that reconstructs a 3D scene from multi-view images. It differs from standard differentiable rendering by explicitly managing the Gaussian population through densification, pruning, and splitting operations during the optimization process, rather than optimizing a fixed set of primitives. Choose this as the foundational training loop for standard 3DGS.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
A strategy for computing gradients that shifts the computational granularity from per-pixel to per-splat (per-Gaussian). It solves the high atomic contention and false sharing inherent in standard per-pixel backpropagation. Choose this to accelerate training time while maintaining numerical equivalence to the standard approach.

### [Grendel-GS](7349-grendel-gs.md)
A distributed optimization framework that partitions the global set of Gaussian primitives across multiple GPUs. It enables the training of scenes with billions of Gaussians by allowing each GPU to render its local subset and combining results via inter-GPU communication. Choose this when single-device memory limits prevent training large scenes. Unlike [TideGS](7350-tidegs.md), which streams data from disk on a single GPU, Grendel-GS distributes the parameter store across a multi-GPU cluster.

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Details specific hyperparameters, data structures, and optimization strategies (e.g., corrected scene extent, half-precision stability, SH degree reduction) required for effective implementation. Choose this to resolve common implementation pitfalls such as scene extent bias and redundant SH computations.

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1]. The initial implementation used a per-pixel parallel backward pass, which suffered from high atomic contention on GPUs. Subsequent work identified that per-splat parallelization reduces this contention [S1]. The need for distributed training arose as scene sizes grew beyond single-GPU memory capacities, leading to frameworks like Grendel-GS [S1, unofficial]. Variants like MaskGaussian introduced probabilistic masks for better pruning, requiring specialized gradient computations [S1]. Implementation details such as corrected scene extent calculation were identified to prevent bias in Gaussian densification [S3].

## Sources
[S1] Kerbl, B., Kopanas, G., Leimkühler, T., & Drettakis, G. (2023). 3D Gaussian Splatting for Real-Time Radiance Field Rendering. ACM Transactions on Graphics (TOG).
[S2] Luiten, J., et al. (2023). Dynamic 3D Gaussians: Tracking by Persistent Dynamic View Synthesis.
[S3] Huang, B., et al. (2023). NSFF: Neural Sparse Voxel Fields for Novel View Synthesis.
[S4] Li, Z., et al. (2023). 4D Gaussian Splatting for Real-Time Dynamic Scene Rendering.
[S6] Wang, P., et al. (2024). Hard Gaussian Growing for Efficient 3D Gaussian Splatting.
[S7] Unofficial implementation notes on SH degree and background color optimization.
