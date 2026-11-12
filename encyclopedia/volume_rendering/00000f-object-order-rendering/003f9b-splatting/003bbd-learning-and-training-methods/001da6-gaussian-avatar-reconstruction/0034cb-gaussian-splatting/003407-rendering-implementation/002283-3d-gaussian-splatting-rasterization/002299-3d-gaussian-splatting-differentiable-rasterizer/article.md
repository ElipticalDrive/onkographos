# 3D Gaussian Splatting Differentiable Rasterizer

Also called: 3D Gaussian Splatting (3DGS) Rendering, 3DGS rasterization pipeline, gsplat rasterization

## Overview

The 3D Gaussian Splatting Differentiable Rasterizer is a screen-space rendering method that projects a set of 3D Gaussian primitives onto a 2D image plane and accumulates color per pixel using alpha compositing, allowing gradients to flow back to Gaussian parameters (position, scale, rotation, opacity, color) and camera view matrices [S5, S6, unofficial]. It differs from its siblings by performing splatting and alpha compositing directly in image coordinates without using OpenGL NDC as an intermediate representation, and by supporting full differentiability with respect to camera pose optimization [S6, unofficial]. It builds on [3D Gaussian Splatting Rasterization](2812-3d-gaussian-splatting-rasterization.md) by providing the specific forward and backward computational graphs required for end-to-end training [S3, S6, unofficial].

## How it works

### Inputs and Outputs

**Inputs:**
*   `means3D`: 3D positions of $N$ Gaussians, shape $(N, 3)$ [S6, unofficial].
*   `quats`: Unit quaternions for rotation, shape $(N, 4)$ [S6, unofficial].
*   `scales`: Log-scale factors, shape $(N, 3)$ [S6, unofficial].
*   `opacities`: Opacity values, shape $(N,)$ [S6, unofficial].
*   `colors`: Pre-computed RGB colors or Spherical Harmonics coefficients [S4, S6, unofficial].
*   `viewmatrix`: $4 \times 4$ camera view matrix [S6, unofficial].
*   `projmatrix`: Camera projection/intrinsics matrix $K$ ($3 \times 3$) or combined view-projection [S6, unofficial].
*   `image_width`, `image_height`: Integer dimensions of the output image [S6, unofficial].
*   `background`: Background color tensor, shape $(3,)$ [S3, S4, unofficial].

**Outputs:**
*   `rendered_image`: RGB image, shape $(H, W, 3)$ [S4, S5, unofficial].
*   `rendered_opacity`: Alpha mask, shape $(H, W)$ [S4, unofficial].
*   `radii`: Radius of each Gaussian in pixels, shape $(N,)$ [S4, unofficial].
*   `num_rendered`: Number of Gaussians contributing to the image [S4, unofficial].

### Data Structures

The rasterizer typically operates on tiled buffers to optimize memory access. A common configuration bins Gaussians into $16 \times 16$ tiles [S6, unofficial].

*   **Gaussian Attributes:** Stored in contiguous arrays or PyTorch tensors on the GPU.
    *   `means3D`: $(N, 3)$ float32.
    *   `cov3D_precomp`: Pre-computed 3D covariance matrices $(N, 6)$ or derived from `quats`/`scales` [S4, S6, unofficial].
    *   `colors`: $(N, C)$ where $C$ depends on SH degree or pre-computed RGB [S4, unofficial].
    *   `opacities`: $(N,)$ float32.
*   **Rasterization Context (CUDA/C++):**
    *   `geomBuffer`: Stores bounding box indices and tile assignments [S4, S6, unofficial].
    *   `binningBuffer`: Stores the sorted list of Gaussian IDs per tile [S4, S6, unofficial].
    *   `imgBuffer`: Temporary storage for per-pixel accumulation during rasterization [S4, unofficial].
    *   `ctx->saved_data`: Stores tensors required for the backward pass: `xys` (2D means), `conics` (inverse 2D covariance), `colors`, `opacity`, `background`, `cov2d`, `camDepths`, `finalTs` (transmittance), `px2gid` (pixel-to-Gaussian ID map) [S3, unofficial].

### Steps

1.  **Preprocessing:** Convert `quats` and `scales` into 3D covariance matrices $\hat{\Sigma}$ [S6, unofficial]. Transform 3D means to camera space using the view matrix [S6, unofficial].
2.  **Projection:** Project 3D camera-space centers to 2D pixel coordinates. Using the affine approximation of the projection Jacobian $J$ at pose $P$:
    $$ J = \frac{1}{d} \begin{bmatrix} 1 & 0 & -\tilde{\mu}_x/d \\ 0 & 1 & -\tilde{\mu}_y/d \end{bmatrix} $$
    where $d$ is the depth and $\tilde{\mu}$ are the camera-space coordinates [S6, unofficial].
    Compute the 2D covariance in image coordinates:
    $$ \Sigma_n^{2D} = J^\top R^\top \Sigma R J $$
    where $R$ is the rotation part of the 3D covariance transformation [S6, unofficial]. Map to pixel coordinates [S6, unofficial].
3.  **Bounding Box & Binning:** For each Gaussian, compute the axis-aligned bounding box (AABB) around the 99th percentile ellipse (3 standard deviations) of the 2D projected covariance [S6, unofficial]. Bin Gaussians into $16 \times 16$ tiles if their AABB intersects the tile [S6, unofficial].
4.  **Sorting:** Sort Gaussians within each tile by depth (back-to-front) using a tile-sorting algorithm [S6, unofficial].
5.  **Rasterization (Forward):**
    *   Initialize canvas `outImg` to `background` and transmittance buffer `T` to ones (shape $H \times W$) [S5, S6, unofficial].
    *   Iterate through Gaussians in sorted order. For each pixel $p_{(x,y)}$ covered by the Gaussian's bounding box:
        *   Compute displacement $d = p_{(x,y)} - \mu_n$ [S6, unofficial].
        *   Compute exponent: $power = -0.5 \cdot d^\top (\Sigma_n^{2D})^{-1} d$ [S5, S6, unofficial].
        *   Compute opacity: $\alpha_n = o_n \cdot \exp(power)$ [S6, unofficial]. Clamp $\alpha_n$ to a maximum of 0.99 to prevent division by zero in gradients [S5, unofficial].
        *   Update transmittance: $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$, with $T_1 = 1$ [S6, unofficial].
        *   Update color: $C_{out} = C_{out} + c_n \cdot \alpha_n \cdot T_i$ [S5, S6, unofficial].
        *   Update transmittance for next Gaussian: $T_{i+1} = T_i \cdot (1 - \alpha_n)$ [S5, unofficial].
    *   Compute final alpha: $\alpha_{out} = 1 - T_{final}$ [S3, unofficial].
6.  **Backward Pass:**
    *   Compute gradients of the loss with respect to `outImg` and `outAlpha` [S3, unofficial].
    *   Propagate gradients through the alpha compositing loop to compute gradients for $\alpha_n$ [S6, unofficial].
    *   Compute gradients for Gaussian parameters (means, covariances, opacity, color) using the chain rule [S6, unofficial].
    *   If `backward_geometry` is enabled, compute gradients for the camera view matrix $P=[R|t]$ [S4, S6, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def rasterize_forward(means3D, quats, scales, opacities, colors, viewmatrix, projmatrix, width, height, background):
    """
    Forward pass of the 3D Gaussian Splatting Differentiable Rasterizer.
    Simplified for clarity; assumes CUDA kernel calls for performance.
    """
    # 1. Transform to camera space
    # R_cam, t_cam = decompose_view_matrix(viewmatrix)
    # means_cam = (R_cam @ means3D.T).T + t_cam
    
    # 2. Project to 2D and compute covariances
    # This involves computing the Jacobian J and transforming the 3D covariance
    # cov2d = compute_2d_covariance(means_cam, cov3d, projmatrix)
    # means2d = project_to_pixel(means_cam, projmatrix)
    
    # 3. Binning and Sorting
    # tiles = bin_gaussians(means2d, cov2d, width, height, tile_size=16)
    # sorted_gaussians = sort_gaussians_by_depth(tiles, means_cam)
    
    # 4. Rasterization Loop (Conceptual Python equivalent of CUDA kernel)
    out_img = background.unsqueeze(0).unsqueeze(0).repeat(1, 1, height, width) # (1, 3, H, W)
    transmittance = torch.ones((1, 1, height, width), device=means3D.device)
    
    # In practice, this is done in parallel tiles
    for tile in tiles:
        for gaussian in sorted_gaussians[tile]:
            # Compute alpha for pixels in tile
            # alpha = compute_alpha(means2d[gaussian], cov2d[gaussian], opacities[gaussian], tile_pixels)
            
            # Update image and transmittance
            # out_img = out_img + colors[gaussian] * alpha * transmittance
            # transmittance = transmittance * (1 - alpha)
            
            pass 
            
    return out_img, 1 - transmittance

def rasterize_backward(out_img_grad, out_alpha_grad, saved_tensors):
    """
    Backward pass to compute gradients for Gaussian parameters.
    """
    # Retrieve saved tensors: means3D, quats, scales, opacities, colors, 
    # cov2d, means2d, transmittance, etc.
    
    # Propagate gradients through the compositing loop
    # alpha_grad = compute_alpha_gradient(out_img_grad, out_alpha_grad, transmittance, colors)
    
    # Compute gradients for means, covs, opacity, color
    # grad_means = compute_mean_gradient(alpha_grad, cov2d, means2d)
    # grad_covs = compute_cov_gradient(alpha_grad, means2d)
    # grad_opac = compute_opacity_gradient(alpha_grad)
    # grad_colors = compute_color_gradient(alpha_grad, transmittance)
    
    # If camera optimization is enabled:
    # grad_viewmatrix = compute_camera_gradient(alpha_grad, means_cam)
    
    return grad_means, grad_quats, grad_scales, grad_opacities, grad_colors
```

### Parameters

*   `tile_size`: Typically $16 \times 16$ pixels [S6, unofficial].
*   `sigma` (variance adjustment): In some variants, a small adjustment factor $\theta = 0.05$ is added to ensure normalization for interior splats [S1].
*   `alpha_max`: Clamping value for opacity, typically 0.99, to prevent numerical instability in gradients [S5, unofficial].
*   `lambda_dssim`: Weight for SSIM term in loss function [S4, unofficial].
*   `lambda_opacity`: Weight for opacity mask loss [S4, unofficial].
*   `lambda_normal_render_depth`: Weight for normal consistency loss, typical value 0.01 [S4, unofficial].

### Edge Cases and Pitfalls

*   **Undefined Alpha Gradient:** If the alpha channel is not used in the loss, `v_outAlpha` may be undefined. It must be initialized to `zeros_like(finalTs)` before backward computation [S3, unofficial].
*   **Memory Constraints:** Iterating over all Gaussians in a Python loop is slow and memory-intensive for large $N$. Production implementations use CUDA kernels [S5, unofficial].
*   **Boundary Normalization:** Derivatives of splat weights do not account for resampling and need to be normalized by accumulated weights. An adjustment factor $\theta$ ensures normalization occurs even if weights sum < 1 at boundaries [S1].
*   **Viewing Axis Elongation:** Standard rasterization does not account for elongation along the viewing axis, causing Gaussians to over-stretch in textureless regions. Isotropic regularization may be required [S9, unofficial].

## Subtopics

### [Gaussian Splat Rasterization (Vertex Shader)](3253-gaussian-splat-rasterization-vertex-shader.md)
This method implements the vertex shader stage of a GPU-based rasterizer for 3D Gaussian Splatting, transforming 3D Gaussian primitives into 2D screen-space ellipses for instanced rendering. It differs from fragment-based or compute-shader approaches by performing all geometric projection, covariance transformation, and ellipse basis computation within the vertex shader, passing only the final vertex positions and attributes to the rasterizer. It builds on [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) by handling the specific data unpacking and frustum culling required for efficient vertex-driven splatting.

### [Masked-Rasterization](4366-masked-rasterization.md)
Masked-Rasterization is a technique that allows probabilistically existing Gaussians to receive gradients during the blending process, even when they are masked out of the final image. It solves the problem of "stale" Gaussians in pruning pipelines by ensuring that unsampled entities can still update their existence probability based on their virtual contribution, rather than being permanently discarded due to zero alpha filtering. It differs from sibling methods like [Analytic-Splatting](3025-analytic-splatting.md) by applying masks during the blending stage rather than modifying Gaussian attributes (opacity/scale) prior to rasterization, which prevents the loss of gradient flow.

### [Differentiable Radiative Rasterization](4526-differentiable-radiative-rasterization.md)
Differentiable Radiative Rasterization (DRR) is a GPU-friendly rasterization process implemented in CUDA that renders novel X-ray projections from radiative Gaussian point clouds. It solves the problem of differentiable rendering for X-ray imaging by projecting 3D Gaussians to a 2D detector plane and accumulating radiation intensity per pixel, allowing gradients to flow back to Gaussian parameters (position, covariance, opacity, and radiation intensity). Unlike standard optical Gaussian Splatting which uses alpha compositing for color, DRR sums the radiation intensities of intersection points weighted by the Gaussian probability density, tailored for the physics of X-ray attenuation.

### [3D/2D Hybrid Gaussian Rasterization with Fallback Sigma](4812-3d-2d-hybrid-gaussian-rasterization-with-fallback-sigma.md)
This method optimizes the rasterization of 3D Gaussians by dynamically selecting the tighter bounding sigma between a 3D ray-surfel intersection estimate and a pre-computed 2D screen-space fallback. It solves the problem of inefficient culling and inaccurate bounding boxes for Gaussians that are far from the camera or highly anisotropic, ensuring that only relevant splats are processed per pixel. It differs from standard 3D Gaussian Splatting rasterization [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) by introducing a hybrid sigma calculation that combines 3D geometric bounds with 2D pre-computed means to improve culling accuracy.

### [Opacity Rendering with SH Colors](7480-opacity-rendering-with-sh-colors.md)
This method performs a rendering pass that uses the model's actual opacity values and precomputes colors from Spherical Harmonics (SH) in Python before rasterization, allowing the rasterizer to use precomputed colors. It differs from standard SH rendering by performing the SH-to-RGB conversion in the Python host code rather than precomputing colors for the rasterizer, allowing the rasterizer to use simpler color inputs while deferring the directional color evaluation to the CPU/GPU Python layer. The source description states that colors are precomputed from SH, but the provided code initializes the color tensor to ones, indicating a potential implementation simplification or external precomputation step.

### [3D Gaussian Unscented Transform (3DGUT)](9154-3d-gaussian-unscented-transform-3dgut.md)
3D Gaussian Unscented Transform (3DGUT) is a rasterization-based rendering technique for 3D Gaussian particles that replaces the standard Elliptical Weighted Average (EWA) splatting formulation with the Unscented Transform (UT). It solves the problem of approximation errors and Jacobian derivation complexity inherent in linearizing non-linear camera projections (e.g., distorted cameras, rolling shutter) by approximating the 3D Gaussian particle itself using sigma points. Unlike its sibling [3D Gaussian Splatting Differentiable Rasterizer](2813-3d-gaussian-splatting-differentiable-rasterizer.md) which relies on first-order Jacobian approximations, 3DGUT enables exact projection under arbitrary non-linear camera models while retaining rasterization efficiency and aligning the rendering formulation with ray-tracing methods to support secondary rays.

### [RoofGS Quadratic Form Expansion and Coefficient Folding](10258-roofgs-quadratic-form-expansion-and-coefficient-folding.md)
RoofGS Quadratic Form Expansion and Coefficient Folding is a rasterization optimization that expands the Gaussian quadratic form to separate pixel-dependent terms from Gaussian-dependent terms, allowing pixel coordinates to be hoisted out of the primitive loop and reducing instruction overhead in the innermost rendering loop. It sets itself apart from standard 3D Gaussian Splatting rasterization by pre-computing and folding opacity and scaling factors into the quadratic coefficients, enabling a streamlined FMA (Fused Multiply-Add) sequence for exponent evaluation.

### [Transmittance of Gaussian Splatting](10263-transmittance-of-gaussian-splatting.md)
This method establishes that the image formation model of 3D Gaussian Splatting (alpha-blending) is mathematically equivalent to a ray-marching volumetric rendering model under specific assumptions. It defines a continuous transmittance function and a view-dependent attenuation coefficient that, when integrated along a ray, produce the same color output as the standard discrete alpha-compositing rasterizer. It sets itself apart from sibling methods by providing the theoretical bridge between the discrete rasterization pipeline and continuous volume rendering, enabling density-based optimizations or alternative sampling strategies that are provably consistent with the original rendering.

### [Rendering Pipeline and Configuration](11406-rendering-pipeline-and-configuration.md)
This family defines the configuration structures and input validation logic required to execute the differentiable rasterization of 3D Gaussian Splatting. It solves the problem of managing mutually exclusive representations (e.g., Spherical Harmonics vs. pre-computed RGB, parametric scales vs. pre-computed covariances) and encapsulating camera parameters into a single context for the CUDA backend. It sets itself apart from the core rasterization algorithm by focusing on data integrity and parameter bundling rather than the geometric projection or compositing steps themselves.

### [Differentiable Backpropagation Mechanisms](11407-differentiable-backpropagation-mechanisms.md)
This family defines the mathematical and algorithmic mechanisms for computing gradients of the rendering loss with respect to 3D Gaussian parameters (position, scale, rotation, opacity, color) during the training of Gaussian Splatting models. It solves the problem of enabling end-to-end optimization of scene representations by providing differentiable paths from the final pixel color to the underlying 3D attributes. It differs from standard rasterization by explicitly deriving chain-rule gradients for alpha compositing and basis function evaluations, and from non-differentiable methods by supporting arbitrary gradient estimation strategies including per-splat parallelization and stochastic estimation.

### Analytic Tau Rasterization
A rasterization branch (`use_analytic_tau`) that computes the per-pixel accumulated analytic line-integral optical thickness $\tau$ along the view ray. Alpha is derived physically as $\text{alpha} = 1 - \exp(-\tau)$ using Beer-Lambert extinction, rather than using heuristic alpha values [S7, unofficial].

### Gaussian Rasterization Kernel Fusion
Skip the concatenation of the two SH coefficient buffers (view-independent and view-dependent bands) and pass these buffers separately to the rasterization backend, fusing the concatenation into the rasterization kernel. Fuse the activation functions for the scales, rotations, and opacities of the Gaussians into the rasterizer to avoid PyTorch-related overhead. Fuse the gradient computations required for the added benefits during training [S8, unofficial].

### Isotropic Loss
A regularisation term applied during RGB-D optimization to prevent Gaussians from over-stretching along the viewing axis, particularly in textureless regions. Standard rasterisation does not account for elongation along the viewing axis, causing Gaussians to over-stretch without isotropic regularisation. On TUM RGB-D, ATE RMSE changes from 1.43 cm (w/o $E_{iso}$) to 1.47 cm (Ours). On Replica RGB-D, Ours achieves 2.25 cm average ATE RMSE. On TUM RGB-D, isotropic regularisation yields only marginal performance difference. On Replica RGB-D, it significantly improves camera tracking [S9, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(N)$ for projection and sorting, where $N$ is the number of Gaussians. Rasterization is $O(P \cdot K)$ where $P$ is the number of pixels and $K$ is the average number of Gaussians per pixel.
*   **Memory Complexity:** $O(N)$ for storing Gaussian attributes. $O(H \cdot W)$ for the image and transmittance buffers. Tile-based sorting requires additional memory for binning buffers [S6, unofficial].
*   **Performance:** The original implementation uses CUDA kernels for real-time performance. Python/PyTorch loops are interpreter-bound and slow, suitable only for pedagogical purposes [S5, unofficial].
*   **Strengths:** Differentiable with respect to camera pose, enabling pose optimization [S6, unofficial]. Efficient tile-based sorting allows real-time rendering of millions of Gaussians [S6, unofficial].
*   **Limitations:** Memory-intensive for very large scenes if not chunked. Python implementations are not suitable for production [S5, unofficial].
*   **When to use:** When training 3D Gaussian Splatting models with gradient-based optimization of scene and camera parameters.
*   **When not to use:** When real-time rendering on CPU is required without CUDA support, or when memory constraints prevent storing all Gaussian attributes in GPU memory.

## Background

The 3D Gaussian Splatting Differentiable Rasterizer originates from the 3D Gaussian Splatting (3DGS) method by Kerbl et al. (2023) [S2][S6, unofficial]. It was developed to provide a differentiable rendering pipeline for optimizing 3D Gaussian primitives from 2D images. The method has been extended in various libraries such as `gsplat` to support full differentiability with respect to camera view matrices and to optimize performance through CUDA kernel fusion [S6, unofficial]. It is widely used in 3D reconstruction, novel view synthesis, and SLAM systems [S9, unofficial].

## Sources

[S1] Differentiable Surface Rendering via Non-Differentiable Sampling, https://doi.org/10.1109/iccv48922.2021.00603, fetched 2026-10-26
[S2] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S3] OpenSplat – 3D gaussian splatting in C++ with CPU and GPU support, https://github.com/pierotofy/OpenSplat, fetched 2026-10-26 (unofficial)
[S4] ldyken53/VEG, https://github.com/ldyken53/VEG, fetched 2026-10-26 (unofficial)
[S5] metin-yat/mini-splat, https://github.com/metin-yat/mini-splat, fetched 2026-10-28 (unofficial)
[S6] gsplat: An Open-Source Library for Gaussian Splatting, https://doi.org/10.48550/arXiv.2409.06765, fetched 2026-10-29 (unofficial)
[S7] hsiang0117/4DGS-Volume-Cloud, https://github.com/hsiang0117/4DGS-Volume-Cloud, fetched 2026-10-26 (unofficial)
[S8] Faster-GS: Analyzing and Improving Gaussian Splatting Optimization, https://arxiv.org/abs/2602.09999v1, fetched 2026-10-26 (unofficial)
[S9] Gaussian Splatting SLAM, https://arxiv.org/abs/2312.06741, fetched 2026-10-27 (unofficial)
