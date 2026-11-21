# FastGS

Also called: FastGS framework, Multi-View Consistent Densification and Pruning (VCD/VCP)

## Overview
FastGS is a general acceleration framework for 3D Gaussian Splatting (3DGS) that significantly reduces training time by regulating the number of Gaussians based on multi-view consistency, dispensing with explicit budgeting mechanisms [S2, unofficial]. It re-engineers the training loop using dynamic sparse rasterization and fast gradient checkpointing to skip negligible computations [S1, unofficial]. Unlike sibling methods that rely on Gaussian-associated properties (Taming-3DGS) or indirect gradient approximations (Speedy-Splat), FastGS directly evaluates each Gaussian's contribution to multi-view reconstruction quality to guide densification and pruning [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Sparse Structure-from-Motion (SfM) point cloud, multi-view images.
*   **Outputs:** Optimized 3D Gaussian parameters (means, covariances, opacities, spherical harmonic coefficients).

**Data Structures:**
Each Gaussian primitive $G_i$ is parameterized as follows [S2, unofficial]:
*   Mean $\mu_i \in \mathbb{R}^3$
*   Rotation $r_i \in \mathbb{R}^4$ (quaternion)
*   Scale $s_i \in \mathbb{R}^3$
*   Opacity $\sigma_i \in \mathbb{R}$
*   Color coefficients $c_i \in \mathbb{R}^{16 \times 3}$ (view-dependent spherical harmonics, degree 3)

The covariance matrix is defined as $\Sigma_i^{3D} = R_i S_i S_i^\top R_i^\top$, where $R_i$ and $S_i$ are rotation and scale matrices derived from $r_i$ and $s_i$ [S2, unofficial].

**Rendering and Projection:**
To render, Gaussians are projected into the 2D image plane. Given a viewing transformation $W$, the covariance in camera coordinates is computed as [S2, unofficial]:
$$ \Sigma_i^{\prime 3D} = J W \Sigma_i^{3D} W^\top J^\top $$
where $J$ is the Jacobian of the affine approximation of the projective transformation. The projected 2D covariance $\Sigma_i^{2D}$ is obtained by marginalizing $\Sigma_i^{\prime 3D}$ along the viewing direction. The 2D Gaussian contribution to pixel $p$ is [S2, unofficial]:
$$ G_i^\prime(p) = \exp\left(-\frac{1}{2} (p - \mu_i^{2D})^\top (\Sigma_i^{2D})^{-1} (p - \mu_i^{2D})\right) $$
Color is accumulated via alpha blending [S2, unofficial]:
$$ C(p) = \sum_{i \in \mathcal{N}} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j), \quad \alpha_i = \sigma_i G_i^\prime(p) $$

**Algorithm Steps:**
1.  **Initialization:** Initialize 3D Gaussians using SfM point clouds [S2, unofficial].
2.  **Dynamic Sparse Rasterization:** During rendering, skip tiles with negligible transmittance to reduce computation [S1, unofficial].
3.  **Gradient Checkpointing:** Use a fast gradient checkpointing scheme that recomputes only necessary intermediate values during the backward pass to save memory [S1, unofficial].
4.  **Multi-View Score Computation (VCD/VCP):**
    *   Sample training views and generate per-pixel L1 loss maps between ground truth and rendered images [S2, unofficial].
    *   For each sampled view, compute a multi-view score for each Gaussian by counting the number of high-error pixels within its 2D footprint [S2, unofficial].
    *   High-error pixels are identified solely from the per-pixel L1 loss [S2, unofficial].
5.  **Adaptive Densification (VCD):** Use the multi-view score to guide densification. If a Gaussian contributes beneficially to multi-view rendering quality (low error contribution), it is retained or densified. This avoids the redundancy seen in methods using Gaussian-associated scores [S2, unofficial].
6.  **Adaptive Pruning (VCP):** Remove Gaussians that are useless to multi-view rendering quality (high error contribution or low importance) [S2, unofficial].
7.  **Optimization:** Update parameters using the accumulated gradients. The method does not require a global budget mechanism for the number of Gaussians [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fastgs_train(sfm_points, multi_view_images, num_iterations):
    # Initialize Gaussians from SfM
    gaussians = initialize_from_sfm(sfm_points)
    
    for iteration in range(num_iterations):
        # 1. Sample views and render
        sampled_views = sample_views(multi_view_images)
        rendered_colors = []
        gt_colors = []
        
        for view in sampled_views:
            # Dynamic sparse rasterization skips low-transmittance tiles
            color = render_dynamic_sparse(gaussians, view.camera)
            rendered_colors.append(color)
            gt_colors.append(view.gt_image)
            
        # 2. Compute Multi-View Scores (VCD/VCP)
        # Calculate L1 loss maps
        loss_maps = compute_l1_loss(rendered_colors, gt_colors)
        
        # Compute importance score for each Gaussian based on high-error pixels in footprint
        gaussian_scores = compute_multi_view_scores(gaussians, loss_maps)
        
        # 3. Adaptive Densification (VCD)
        # Densify Gaussians that have low error contribution (high importance)
        # but are currently sparse or small
        gaussians = densify_vcd(gaussians, gaussian_scores, threshold_low_error)
        
        # 4. Adaptive Pruning (VCP)
        # Remove Gaussians with high error contribution (low importance)
        gaussians = prune_vcp(gaussians, gaussian_scores, threshold_high_error)
        
        # 5. Backward Pass with Gradient Checkpointing
        # Recompute only necessary intermediates to save memory
        gradients = backward_pass_checkpointed(gaussians, sampled_views, gt_colors)
        
        # 6. Optimizer Step
        update_gaussians(gaussians, gradients)
        
    return gaussians
```

**Parameters:**
*   Training time target: ~100 seconds for static scenes (e.g., Tanks & Temples train scene) [S2, unofficial].
*   Loss weights: Standard 3DGS loss weights apply ($\lambda_{SSIM}$, etc.) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Redundancy:** Methods relying on Gaussian-associated scores (like Taming-3DGS) lead to excessive growth of redundant Gaussians because they do not leverage actual contribution to rendering quality [S2, unofficial].
*   **Quality Degradation:** Methods relying on gradients (like Speedy-Splat) lead to significant degradation in rendering quality because they indirectly enforce multi-view consistency [S2, unofficial].
*   **Precision:** Dynamic sparse rasterization requires careful handling of transmittance thresholds to avoid artifacts [S1, unofficial].

## Subtopics

### Accelerated Training for 3DGS
A strategy to improve training time efficiency by separating higher SH bands from the 0th band within the rasterizer, reducing the number of updates for higher SH bands compared to diffused colors [S3]. It modifies the rasterizer to split SH bands from the diffused color, updating SH bands every 16 iterations while updating diffused color at every step [S3]. It also optimizes SSIM loss calculation by replacing the standard 11x11 2D Gaussian kernel with two smaller 1D Gaussian kernels, calculated via a fused CUDA kernel [S3]. The total loss is defined as $L = (1 - \lambda_{SSIM})L_1 + \lambda_{SSIM} L_{D-SIM} + \lambda_m L_m + \lambda_{sh} L_{sh}$ [S3].

### 128-bit Splat Data Structure
Restructuring splat attributes (color, bounding box, position) into 128-bit aligned vectors to utilize optimized memory load instructions and reduce the total number of memory operations [S4]. The new structure packs: Field 1: Color (4D float vector, 128 bits, padded from original 96-bit 3D float); Field 2: Bounding Box (4D unsigned integer vector, 64 bits, 16-bit uints in pixel space); Field 3: Screen Position (2D float vector, 64 bits). The Bounding Box and Screen Position are packed to form a single 128-bit structure alongside the Color [S4]. This trades additional compute instructions for unpacking data for further reductions in memory operation instructions [S4]. Using the original 96-bit 3D float vector for color lacks optimized load instructions, leading to inefficient memory access [S4].

## Operating characteristics
*   **Time Complexity:** Significantly reduced training time. Achieves 3.29× faster training than state-of-the-art methods on Mip-NeRF 360 dataset [S2, unofficial]. Achieves 15.45× faster training than vanilla 3DGS on Deep Blending dataset [S2, unofficial].
*   **Memory Complexity:** Reduced via gradient checkpointing and dynamic sparse rasterization [S1, unofficial].
*   **Performance:** Completes training on Tanks & Temples train scene in approximately 100 seconds [S2, unofficial].
*   **Scaling:** Demonstrates 2.82× faster training for dynamic scene reconstruction and 2.24× faster for surface reconstruction [S2, unofficial].
*   **Strengths:** General applicability to static, dynamic, surface, sparse-view, large-scale reconstruction, and SLAM tasks [S2, unofficial]. No budgeting mechanism required [S2, unofficial].
*   **Limitations:** Relies on accurate multi-view consistency; performance may degrade if multi-view data is insufficient or noisy [S2, unofficial].

## Background
FastGS was proposed as a general acceleration framework for 3D Gaussian Splatting to address the trade-off between training time and rendering quality [S2, unofficial]. It was introduced in a preprint in December 2025 [S2, unofficial]. The method builds on the foundational differentiable rasterization and gradient accumulation principles of 3DGS [6233-optimization-and-parallelization]. It specifically targets the inefficiencies of existing densification and pruning strategies by introducing multi-view consistent evaluation metrics [S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] FastGS: Training 3D Gaussian Splatting in 100 Seconds, https://doi.org/10.48550/arxiv.2511.04283, fetched 2026-10-26 (unofficial)
[S3] Trick-GS: A Balanced Bag of Tricks for Efficient Gaussian Splatting, https://doi.org/10.1109/icassp49660.2025.10889395, fetched 2026-10-30
[S4] Warp-Level Culling for Efficient Blending in 3D Gaussian Splatting, https://doi.org/10.1145/3820019, fetched 2026-11-01
