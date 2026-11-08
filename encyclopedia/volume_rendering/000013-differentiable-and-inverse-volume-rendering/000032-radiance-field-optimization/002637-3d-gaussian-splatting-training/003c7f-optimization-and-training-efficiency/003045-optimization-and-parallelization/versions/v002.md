# Optimization and Parallelization

Also called: Distributed Training, Gradient Parallelization, Multi-GPU 3DGS

## Overview
Optimization and Parallelization addresses the computational bottlenecks and memory constraints inherent in training large-scale 3D Gaussian Splatting (3DGS) models. It encompasses strategies for accelerating the backward pass, distributing the parameter store across multiple GPUs, and managing numerical stability during optimization. It differs from standard optimization by focusing on the specific data parallelism and memory access patterns of explicit Gaussian primitives, rather than implicit neural field updates.

## Shared principles
The training loop relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters (e.g., masks).

The forward pass computes the pixel color $\mathbf{c}(\mathbf{x})$ for each pixel $\mathbf{x}$ by alpha-blending Gaussians sorted by depth. The backward pass computes gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all parameters $\theta$.

Key shared mathematical components include:
1.  **Alpha Blending**: The color contribution of a Gaussian $i$ is modulated by its opacity $\alpha_i$ and transmittance $T_i$ from preceding Gaussians.
2.  **Gradient Propagation**: Gradients flow from the loss through the pixel color, through the alpha-blending chain, to the individual Gaussian attributes.
3.  **Parallelization Granularity**: Computation is parallelized across pixels, splats (Gaussians), or batches of views. The standard approach is per-pixel backpropagation, but alternatives exist [S2, unofficial].
4.  **Memory Management**: Large scenes require managing the Gaussian list in memory, potentially across multiple GPUs or via out-of-core techniques.

## Subtopics

### [MaskGaussian Gradient Computation](4254-maskgaussian-gradient-computation.md)
A method for computing the gradient of a loss function $\mathcal{L}$ with respect to per-Gaussian probabilistic masks $\mathcal{M}_i$. It solves the problem of optimizing mask parameters that modulate Gaussian opacity, enabling adaptive Gaussian representation and pruning. It differs from standard 3DGS backpropagation by explicitly accounting for the interaction between the mask-modulated Gaussian contribution and the background color contribution, which is modulated by the final transmittance.

### [Training Pipeline](5325-training-pipeline.md)
The iterative optimization process that minimizes a differentiable loss function between rendered images and ground-truth views by updating Gaussian parameters. It distinguishes itself from standard neural field optimization by using a differentiable rasterizer to compute gradients directly on explicit geometric primitives (Gaussians) rather than implicit neural networks, allowing for adaptive control (pruning, cloning, splitting) of the scene representation during training.

### [Parallelization for Backpropagation](5989-parallelization-for-backpropagation.md)
A strategy for computing gradients in 3D Gaussian Splatting that shifts the computational granularity from per-pixel to per-splat (per-Gaussian). It solves the high atomic contention and false sharing inherent in standard per-pixel backpropagation, where thousands of threads simultaneously attempt to update the same Gaussian parameters. By restructuring the backward pass, it significantly accelerates training time while maintaining numerical equivalence to the standard approach.

### [Grendel-GS](7349-grendel-gs.md)
A distributed optimization framework for 3D Gaussian Splatting that partitions the global set of Gaussian primitives across multiple GPUs to overcome single-device memory limits. It enables the training of scenes with billions of Gaussians by allowing each GPU to render its local subset and combining results via inter-GPU communication at partition boundaries. Unlike [TideGS](7350-tidegs.md), which streams data from disk on a single GPU, Grendel-GS distributes the parameter store across a multi-GPU cluster.

### [3D Gaussian Splatting Implementation Details](9128-3d-gaussian-splatting-implementation-details.md)
Details the specific software engineering choices, hyperparameters, and numerical stability techniques required to implement the 3D Gaussian Splatting (3DGS) training pipeline efficiently. It addresses the gap between the theoretical differentiable rendering formulation and a production-ready CUDA implementation, focusing on precision management, library selection, and scene initialization corrections. It sets itself apart from general optimization articles by providing concrete values for loss weights, optimizer settings, and memory fetch strategies that directly impact training convergence and hardware utilization.

## Background
The optimization of 3D Gaussian Splatting has evolved from the original implementation [S1] to include specialized techniques for large-scale scenes and specific regularization needs. The introduction of per-splat parallelization [S1] and distributed frameworks like Grendel-GS [S1, unofficial] addresses the scalability limits of single-GPU training. Implementation details regarding numerical stability and initialization [S1, S2][S4, unofficial] have become critical for robust convergence in diverse environments.

## Sources
