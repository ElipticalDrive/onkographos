# Optimization and Training Efficiency

Also called: Training Efficiency, Optimization Speedup

## Overview
This family addresses the computational bottlenecks of training 3D Gaussian Splatting (3DGS), specifically the high memory footprint of large Gaussian populations and the latency of the differentiable rasterization backward pass. It solves the problem of scaling training to large-scale scenes and complex geometries by distributing parameters across multiple GPUs, restructuring gradient computation for hardware efficiency, and optimizing low-level kernel execution. It differs from standard differentiable rendering pipelines by explicitly managing the dynamic population of Gaussians (densification/pruning) in parallel with the optimization loop, and by utilizing specialized CUDA kernels for alpha blending rather than generic automatic differentiation [S2][S3, unofficial].

## Shared principles
The training process relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters.

Key shared mathematical components include:
1.  **Alpha Blending**: The color contribution of a Gaussian $i$ is modulated by its opacity $\alpha_i$ and transmittance $T_i$ from preceding Gaussians.
2.  **Gradient Propagation**: Gradients flow from the loss through the pixel color, through the alpha-blending chain, to the individual Gaussian attributes.
3.  **Parallelization Granularity**: Computation is parallelized across pixels, splats (Gaussians), or batches of views. The standard approach is per-pixel backpropagation, but alternatives exist [S2].
4.  **Memory Management**: Large scenes require managing the Gaussian list in memory, potentially across multiple GPUs or via out-of-core techniques.

## Subtopics

### [3D Gaussian Splatting Training > Parameter-Specific Learning Rates](4276-3d-gaussian-splatting-training-parameter-specific-learning-r.md)
Assigns distinct learning rates to different groups of Gaussian attributes (position, scale, rotation, opacity, color) within a single Adam optimizer. This stabilizes training by preventing parameters with high sensitivity (such as color or scale) from diverging due to large gradients, while allowing slower-converging parameters (such as position) to update appropriately. It sets itself apart from standard uniform learning rate approaches by decoupling the update magnitudes per attribute, addressing the varying gradient scales inherent in the differentiable rasterization pipeline.

### [Optimization and Parallelization](6233-optimization-and-parallelization.md)
Addresses the computational bottlenecks of training 3D Gaussian Splatting (3DGS), specifically the high memory footprint of large Gaussian populations and the latency of the differentiable rasterization backward pass. It solves the problem of scaling training to large-scale scenes and complex geometries by distributing parameters across multiple GPUs, restructuring gradient computation for hardware efficiency, and optimizing low-level kernel execution. It differs from standard differentiable rendering pipelines by explicitly managing the dynamic population of Gaussians (densification/pruning) in parallel with the optimization loop, and by utilizing specialized CUDA kernels for alpha blending rather than generic automatic differentiation.

### [CLM](7351-clm.md)
Removes the GPU memory barrier by moving Gaussian states between CPU and GPU memory during optimization. It employs a smart caching policy to keep the most frequently updated Gaussians on the GPU.

### [FastGS](9177-fastgs.md)
Re-engineers the 3D Gaussian Splatting training loop to accelerate optimization by combining dynamic sparse rasterization with fast gradient checkpointing. It sets itself apart from standard training by skipping tiles with negligible transmittance during rasterization and recomputing only necessary intermediate values during the backward pass, rather than storing all activations.

### [Fused Backward and Optimizer](9597-fused-backward-and-optimizer.md)
Integrates the parameter update step of the optimizer directly into the backward pass of the differentiable rasterizer. It solves the problem of high VRAM usage and computational overhead associated with separate optimizer kernels by eliminating the need for intermediate buffers to store parameter states during the gradient computation phase.

### Faster-GS
- **Definition**: Addresses oscillation in gradient updates for scaling and rotation parameters caused by a poorly conditioned learning rate ratio [S1, unofficial].
- **Step**: Rescales the scaling gradient with the inverse of the current scale to stabilize convergence [S1, unofficial].
- **Impact**: Cuts training time by nearly half compared to standard 3DGS [S1, unofficial].

### Reduced Iteration Training
- **Step**: Reduce training iterations from the original 30,000 to 1,000 for the Neural3DVideo dataset [S2]. Reduce training iterations from the original 30,000 to 1,500 for the LongVolumetricVideo dataset [S2]. Keep all other parameters identical to the original 3DGS implementation [S2].
- **Pitfall**: Using the original 30,000 iterations on sparse input data causes overfitting [S2].
- **Impact**: Reduces training time and improves novel view synthesis by approximately 4 PSNR [S2].

### Disk-Based Training Image Loading
- **Definition**: Loading ground truth images from disk every iteration instead of storing them in VRAM, to enable training on consumer hardware with limited VRAM for ultra-large-scale datasets [S3, unofficial].
- **Comparison**: Standard 3DGS stores the entire training dataset in VRAM for speed. This method loads images from disk every iteration, saving VRAM but reducing training performance [S3, unofficial].

### Sampling Gap Control
- **Definition**: Controlling the sampling rate by adjusting the sampling gap value to influence training speed and rendering quality [S4, unofficial].
- **Step**: Increase the sampling gap value to decrease the sampling rate, which is inversely proportional to the sampling rate [S4, unofficial].
- **Pitfall**: Increasing the sampling gap increases pc time and makes the rendering result blurrier [S4, unofficial].
- **Impact**: On the truck dataset with 3000 training steps, increasing the sampling gap from 2 to 500 increases pc time from 0.17739 to 0.20525 [S4, unofficial].

## Background
The original 3D Gaussian Splatting implementation [22] established a baseline training loop that stores the entire training dataset in VRAM and runs for 30,000 iterations. Subsequent research has focused on mitigating the memory and time costs of this approach. Methods like [S3, unofficial] introduced external memory management for ultra-large scale reconstruction, while [S4, unofficial] explored multiple dimensions of optimization including sampling gaps and learning rate adjustments. [S2] demonstrated that reducing iterations on sparse data prevents overfitting and improves quality. [S1, unofficial] highlighted issues with gradient oscillation in scaling/rotation parameters, leading to methods like Faster-GS.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
[S3] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S4] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
