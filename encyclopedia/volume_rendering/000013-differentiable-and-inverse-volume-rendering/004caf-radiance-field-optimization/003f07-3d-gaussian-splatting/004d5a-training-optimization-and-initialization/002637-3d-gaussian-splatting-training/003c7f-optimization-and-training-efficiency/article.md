# Optimization and Training Efficiency

Also called: Training Optimization, Efficient 3DGS Training

## Overview
This family encompasses the computational strategies, memory management techniques, and hyperparameter configurations required to scale 3D Gaussian Splatting (3DGS) training to large scenes and high iteration counts. It addresses the bottleneck of differentiable rasterization and gradient computation by optimizing the training loop infrastructure, memory bandwidth, and convergence stability. It sets itself apart from [Base 3DGS Optimization](7549-base-3d-gaussian-splatting-optimization.md) by focusing on the engineering of the training loop (parallelization, caching, fused kernels) rather than the mathematical formulation of the loss or the specific densification logic.

## Shared principles
The training process relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters.

Key shared mathematical components include:
1.  **Alpha Blending**: The color contribution of a Gaussian $i$ is modulated by its opacity $\alpha_i$ and transmittance $T_i$ from preceding Gaussians.
2.  **Gradient Propagation**: Gradients flow from the loss through the pixel color, through the alpha-blending chain, to the individual Gaussian attributes.
3.  **Parallelization Granularity**: Computation is parallelized across pixels, splats (Gaussians), or batches of views. The standard approach is per-pixel backpropagation, but alternatives exist [S2].
4.  **Memory Management**: Large scenes require managing the Gaussian list in memory, potentially across multiple GPUs or via out-of-core techniques.

## Subtopics

### [3D Gaussian Splatting Training > Parameter-Specific Learning Rates](4276-3d-gaussian-splatting-training-parameter-specific-learning-r.md)
[Parameter-Specific Learning Rates](4276-3d-gaussian-splatting-training-parameter-specific-learning-r.md) assigns distinct learning rates to different groups of Gaussian attributes (position, scale, rotation, opacity, color) within a single Adam optimizer. It stabilizes training by preventing parameters with high sensitivity (such as color or scale) from diverging due to large gradients, while allowing slower-converging parameters (such as position) to update appropriately. Choose this when standard uniform learning rates cause instability or slow convergence in specific attributes.

### [Optimization and Parallelization](6233-optimization-and-parallelization.md)
[Optimization and Parallelization](6233-optimization-and-parallelization.md) encompasses the computational strategies required to scale 3DGS training to large scenes. It addresses the bottleneck of differentiable rasterization by distributing memory across multiple GPUs, optimizing the granularity of parallel execution, and managing the iterative population dynamics of the Gaussian primitives. Choose this for large-scale scenes where standard single-GPU training is infeasible.

### [CLM](7351-clm.md)
[CLM](7351-clm.md) eliminates the performance bottleneck of GPU memory barriers by dynamically migrating Gaussian states between CPU and GPU memory. It solves the problem of excessive data transfer overhead during training by employing a smart caching policy that retains frequently updated Gaussians on the GPU while offloading less active ones to CPU RAM. Choose this when GPU VRAM is insufficient to hold the active working set of Gaussians, but CPU RAM is abundant. It differs from static partitioning methods like Grendel-GS and out-of-core streaming frameworks like TideGS by focusing on temporal locality of updates rather than spatial partitioning or disk-based streaming.

### [FastGS](9177-fastgs.md)
[FastGS](9177-fastgs.md) re-engineers the 3D Gaussian Splatting training loop to accelerate optimization by combining dynamic sparse rasterization with fast gradient checkpointing. It sets itself apart from standard training by skipping tiles with negligible transmittance during rasterization and recomputing only necessary intermediate values during the backward pass, rather than storing all activations. Choose this when VRAM limits prevent large batch sizes or when training time is constrained by memory bandwidth.

### [Fused Backward and Optimizer](9597-fused-backward-and-optimizer.md)
[Fused Backward and Optimizer](9597-fused-backward-and-optimizer.md) integrates the parameter update step of the optimizer (typically Adam) directly into the backward pass of the differentiable rasterizer. It solves the problem of high VRAM usage and computational overhead associated with separate optimizer kernels by eliminating the need for intermediate buffers to store parameter states during the gradient computation phase. Choose this to reduce VRAM footprint and memory bandwidth pressure during the backward pass.

### Faster-GS
Addresses oscillation in gradient updates for scaling and rotation parameters caused by a poorly conditioned learning rate ratio [S1, unofficial]. It rescales the scaling gradient with the inverse of the current scale to stabilize convergence [S1, unofficial]. This cuts training time by nearly half compared to standard 3DGS [S1, unofficial].

### Reduced Iteration Training
Reduces training iterations from the original 30,000 to 1,000 for the Neural3DVideo dataset and to 1,500 for the LongVolumetricVideo dataset [S2]. It keeps all other parameters identical to the original 3DGS implementation [S2]. Using the original 30,000 iterations on sparse input data causes overfitting [S2]. This reduces training time and improves novel view synthesis by approximately 4 PSNR [S2].

### Disk-Based Training Image Loading
Loading ground truth images from disk every iteration instead of storing them in VRAM, to enable training on consumer hardware with limited VRAM for ultra-large-scale datasets [S3, unofficial]. Standard 3DGS stores the entire training dataset in VRAM for speed. This method saves VRAM but reduces training performance [S3, unofficial].

### Sampling Gap Control
Controlling the sampling rate by adjusting the sampling gap value to influence training speed and rendering quality [S4, unofficial]. Increasing the sampling gap value decreases the sampling rate, which is inversely proportional to the sampling rate [S4, unofficial]. Increasing the sampling gap increases pc time and makes the rendering result blurrier [S4, unofficial]. On the truck dataset with 3000 training steps, increasing the sampling gap from 2 to 500 increases pc time from 0.17739 to 0.20525 [S4, unofficial].

## Background
The original 3D Gaussian Splatting implementation [22] established a baseline training loop that stored the entire training dataset in VRAM and used fixed learning rates [S3, unofficial]. As the method gained popularity, researchers identified bottlenecks in memory usage, training speed, and convergence stability for large-scale and dynamic scenes. Methods like Faster-GS [79] addressed gradient oscillations, while others like CLM and FastGS focused on memory management and kernel fusion. Disk-based loading strategies emerged to support ultra-large-scale reconstruction on consumer hardware [S3, unofficial]. Sampling gap control was introduced to balance training speed and quality in specific contexts [S4, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
[S3] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S4] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
