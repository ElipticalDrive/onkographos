# gsplat
Also called: gsplat API

## Overview
gsplat is a unified, high-performance API that continuously integrates low-level scheduling optimizations for constrained environments [S1, unofficial]. It primarily improves the rasterization and training backend rather than changing the scene representation itself [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians defined by position $\mathbf{p}_k$, covariance $\Sigma_k$, opacity $\alpha_k$, and spherical harmonics coefficients for color [G].
*   **Output:** A rasterized 2D image (color and depth) and gradients for training [G].

**Data Structures:**
The method relies on a structured layout of Gaussian attributes to facilitate efficient matrix operations. The core computation uses a matrix multiplication $C = AB$, where $A$ contains the Gaussian attributes and $B$ contains the pixel-wise accumulation weights [S1, unofficial].

**Steps:**
1.  **Attribute Loading:** Load Gaussian attributes from global memory. For constrained environments, only Gaussians visible in the current view frustum are loaded from flash storage using quantized integer representations [S1, unofficial].
2.  **Projection:** Transform 3D Gaussian centers to screen space using the Model-View-Projection (MVP) matrix [G]. Compute the 2D covariance matrix $\Sigma_{2D}$ using the Jacobian of the projection transformation [G].
3.  **Splat Rasterization:** For each tile, determine the contribution of overlapping Gaussians by computing the 2D Gaussian footprint and accumulating weights [G].
4.  **Matrix Multiplication:** Perform $C = AB$ to compute the final color and depth values, leveraging GPU tensor cores or optimized shader instructions [S1, unofficial].
5.  **Compositing:** Accumulate the contributions using alpha blending to handle transparency [G].

**Parameters:**
*   **Tile Size:** The screen is partitioned into finer tiles to optimize cache locality and multi-GPU scheduling [S1, unofficial].
*   **Quantization:** Positions and colors may be quantized to lower precision (e.g., integers) for memory-constrained deployments [S1, unofficial].

**Edge Cases:**
*   **Precision:** Mobile-GS performs blending in a lower-precision shader [S1, unofficial].
*   **Visibility:** Culling Gaussians outside the view frustum is critical for performance in constrained environments [S1, unofficial].

## Subtopics

### Warp-Level Gradient Accumulation
An optimization technique using warp-level reductions in CUDA to efficiently accumulate gradients during training [S3, unofficial]. Implemented in a custom CUDA kernel, it is applied to improve training efficiency in custom multi-view regularization kernels [S3, unofficial].

## Operating characteristics
*   **Performance:** Enables real-time rendering on GPUs and mobile devices. Performance depends on visibility, projected footprint, memory access, and rasterizer implementation rather than Gaussian count alone [S2]. Quantized integer representations and lower-precision shaders reduce memory bandwidth pressure [S1, unofficial].
*   **Storage:** Internal compression mechanisms like vector quantization (replacing Gaussian attributes with codebook entries) can reduce model size from 75.50 to 5.54 MiB (92.66% reduction) while retaining 25.08 dB PSNR [S2].
*   **Scaling:** Smaller representations do not automatically imply faster rendering; a pruned model with one-tenth the Gaussians may yield lower frame rates (155.15 versus 193.46 rFPS) due to memory access patterns [S2].
*   **Strengths:** Unified API, low-level scheduling optimizations, supports constrained environments, accelerates training [S1, unofficial][S2].
*   **Limitations:** Frame rate is sensitive to memory access and rasterizer implementation; aggressive pruning can degrade performance despite reduced Gaussian counts [S2].

## Background
gsplat continuously integrates low-level scheduling optimizations into a unified high-performance API [S1, unofficial]. It underpins variants such as FlashGS, which accelerates forward processing by partitioning the screen into finer tiles for multi-GPU pipelines, and Mobile-GS, which targets smartphone deployment with memory-centric rendering [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting for real-time scene reconstruction: Methods, systems, and trade-offs, https://doi.org/10.1016/j.patcog.2026.115120, fetched 2026-10-26
[S3] Geometry-Grounded Gaussian Splatting, https://arxiv.org/abs/2601.17835v2, fetched 2026-10-26 (unofficial)
