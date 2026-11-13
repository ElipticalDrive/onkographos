# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and algorithmic variants required to train and render 3D Gaussian Splatting (3DGS) scenes effectively. It addresses the core problem of managing the large number of Gaussian primitives, handling view-dependent appearance, and optimizing scene geometry under various constraints (e.g., underwater, sparse views, dynamic scenes). It sets itself apart from general volume rendering by focusing on the specific differentiable rasterization pipeline, adaptive density control, and hardware-aware optimizations (such as half-precision memory fetches) unique to the 3DGS framework [S1, S4, S6].

## How it works

### Inputs and Outputs
**Inputs:**
*   Sparse point cloud from Structure-from-Motion (SfM) [S1, S3].
*   Camera poses and intrinsics [S1, S6].
*   Training images (RGB) [S1, S6].
*   Optional: Depth maps, normal priors, or semantic labels [S1, S10].

**Outputs:**
*   Optimized set of 3D Gaussians, each defined by:
    *   Mean position $\mu \in \mathbb{R}^3$ [S8].
    *   Opacity $\alpha \in [0, 1]$ [S8].
    *   Rotation matrix $R \in \mathbb{R}^{3 \times 3}$ (or quaternion) [S8].
    *   Scaling vector $s \in \mathbb{R}^3$ [S8].
    *   Spherical Harmonics (SH) coefficients for view-dependent color $c$ [S8].
*   Rendered images $\hat{C}(p)$ for novel views [S2, unofficial].

### Data Structures
*   **Gaussian List:** A dynamic array of Gaussian primitives. In standard 3DGS, this is an explicit list [S1]. In voxel-based variants (GVKF), Gaussians are managed within a sparse voxel grid, where each voxel holds a feature vector $F \in \mathbb{R}^d$ that decodes attributes via MLPs [S3].
*   **Voxel Grid (Optional):** For methods like GVKF, a sparse 3D grid manages Gaussian growth and pruning. Each voxel can generate up to $m$ Gaussians [S3].
*   **Illumination Grid (Optional):** For methods like 3iGS, a factorized tensor grid $\mathcal{G}_l$ stores local illumination [S8].
*   **Temporal Trajectory (Optional):** For 4D-GS, per-Gaussian temporal trajectories stored as polynomial coefficients or Radial Basis Function (RBF) parameters [S5, unofficial].

### Numbered Steps

1.  **Initialization:**
    *   Initialize Gaussians from SfM points. The scene extent $e_{\text{scene}}$ is typically calculated based on camera positions, but can be corrected using SfM points to avoid bias [S11].
    *   Set initial SH degree (typically 0 or 1) [S10].
    *   For underwater scenes, initialize backscatter parameters $B_\infty, B_b$ using nonlinear least squares on pixels with RGB values below the 1st percentile [S6].

2.  **Forward Rendering (Rasterization):**
    *   Project 3D Gaussians to 2D screen space using the viewing transformation $W$ and affine approximation $J$ [S8].
    *   Compute 2D covariance $\Sigma' = J W \Sigma W^T J^T$ where $\Sigma = R S S^T R^T$ [S8].
    *   Sort Gaussians by depth for each tile/pixel [S2, unofficial].
    *   Alpha-blend Gaussians to compute pixel color $\hat{C}(p)$:
        $$ \hat{C}(p) = \sum_{i=1}^{M} c_i T_i \alpha_i G_i(p) $$
        where $T_i = \prod_{k=1}^{i-1} (1 - \alpha_k G_k(p))$ and $G_i(p) = \exp(-\frac{1}{2}(p - \mu'_i)^T (\Sigma'_i)^{-1} (p - \mu'_i))$ [S2, unofficial].
    *   For underwater scenes, apply attenuation and backscatter:
        $$ C_{uw} = \sum_{i=1}^M c_i \alpha_i T_i e^{-B_d z} + B_\infty (1 - e^{-B_b z}) $$
        where $z$ is the depth extracted from the scene [S6].
    *   For methods with local illumination (3iGS), compute incoming illumination $L_i$ from the grid and specular color $\mathbf{c_s}$ via a neural network $\mathcal{F}$, then combine with diffuse color $\mathbf{c_d}$: $\mathbf{c}(\omega_o) = \mathbf{c_d} + \mathbf{c_s}(\omega_o)$ [S8].

3.  **Loss Computation:**
    *   Compute loss $\mathcal{L}$ between rendered image $\hat{I}$ and ground truth $I^{gt}$:
        $$ \mathcal{L} = (1 - \lambda) \mathcal{L}_1 + \lambda \mathcal{L}_{\text{D-SSIM}} $$
        with $\lambda = 0.2$ [S8].
    *   For underwater scenes, add backscatter loss $L_{bs}(B_\infty, B_b) = \|B_\infty - \hat{B}_\infty\|_{\ell_1} + \|B_b - \hat{B}_b\|_{\ell_1}$ [S6].
    *   For multi-view training, use mini-batches to reduce gradient variance [S2, unofficial].

4.  **Backpropagation:**
    *   Compute gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all parameters $\theta$ (position, covariance, opacity, color, SH coefficients) [Parent].
    *   For underwater scenes, compute partial derivatives for $B_d, B_\infty, B_b$ [S6].
    *   For 3iGS, optimize BRDF features relative to the illumination field [S8].

5.  **Optimization Step:**
    *   Update parameters using Adam optimizer with learning rate $\eta_t = 1 \times 10^{-3}$ [S10].
    *   For underwater scenes, re-estimate $B_\infty, B_b$ every 500 steps to guide convergence [S6].
    *   For 3iGS, shrink the illumination grid boundary midway through training [S8].

6.  **Adaptive Density Control (ADC):**
    *   Evaluate metrics for each Gaussian: opacity, screen-space size, and gradient magnitude of means [S1].
    *   **Cull:** Remove Gaussians with low opacity or small screen size.
    *   **Split:** Duplicate Gaussians with high gradient magnitude and large screen size.
    *   **Duplicate:** Duplicate Gaussians with high gradient magnitude but small screen size.
    *   Perform these operations at fixed intervals [S1].
    *   *Pitfall:* Standard ADC assumes single-view training. In multi-view training, gradients from different image planes may be incompatible, leading to suboptimal densification [S2, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def train_3dgs(scene_gaussians, images, cameras, num_iterations, lambda_ssim=0.2):
    optimizer = Adam(scene_gaussians.parameters(), lr=1e-3)
    
    for iteration in range(num_iterations):
        # 1. Forward Pass
        rendered_images = []
        for image, camera in zip(images, cameras):
            # Project Gaussians to 2D
            projected_gaussians = project_to_screen(scene_gaussians, camera)
            
            # Sort by depth
            sorted_gaussians = sort_by_depth(projected_gaussians)
            
            # Alpha Blend
            color = 0.0
            transmittance = 1.0
            for g in sorted_gaussians:
                # Compute 2D Gaussian value
                g_val = exp(-0.5 * (g.p - g.mu_prime).T @ inv(g.Sigma_prime) @ (g.p - g.mu_prime))
                alpha = g.alpha * g_val
                color += transmittance * alpha * g.color
                transmittance *= (1 - alpha)
            rendered_images.append(color)
        
        # 2. Loss Computation
        loss = 0.0
        for rendered, gt in zip(rendered_images, images):
            l1_loss = L1(rendered, gt)
            ssim_loss = D_SSIM(rendered, gt)
            loss += (1 - lambda_ssim) * l1_loss + lambda_ssim * ssim_loss
        loss /= len(images)
        
        # 3. Backward Pass
        optimizer.zero_grad()
        loss.backward()
        
        # 4. Optimization Step
        optimizer.step()
        
        # 5. Adaptive Density Control (every N iterations)
        if iteration % 100 == 0:
            for g in scene_gaussians:
                grad_mag = norm(g.gradient_of_mean)
                screen_size = compute_screen_space_size(g)
                opacity = g.opacity
                
                if opacity < threshold_low or screen_size < threshold_small:
                    scene_gaussians.remove(g) # Cull
                elif grad_mag > threshold_high and screen_size > threshold_large:
                    scene_gaussians.split(g) # Split
                elif grad_mag > threshold_high and screen_size <= threshold_large:
                    scene_gaussians.duplicate(g) # Duplicate

    return scene_gaussians
```

### Parameters
*   **Learning Rate:** $1 \times 10^{-3}$ for Gaussian parameters [S10].
*   **Loss Weight ($\lambda$):** 0.2 for D-SSIM [S8].
*   **SH Degree:** Typically 0 or 1 [S10]. Setting to 0 improves training speed by 16.25% [S12, unofficial].
*   **Underwater Parameters:**
    *   $B_d$: Direct attenuation, learnable [S6].
    *   $B_\infty \in [0, 1]$: Infinite backscatter, learnable [S6].
    *   $B_b \in [0, 5]$: Backscatter coefficient, learnable [S6].
    *   $\lambda_2 = 0.1$: Weight for backscatter loss [S6].
*   **3iGS Parameters:**
    *   Illumination grid resolution: $150^3$ voxels [S8].
    *   Rank $R_c$: Tensorial factorization rank [S8].
    *   Training iterations: 30,000 [S8].

### Edge Cases and Pitfalls
*   **Multi-view Training:** Naive multi-view training causes idle threads in the tile-based rasterizer. Use thread-efficient partial rendering [S2, unofficial]. Standard ADC fails in multi-view because gradients from different planes are incompatible [S2, unofficial].
*   **Underwater Images:** Camera sensor must have a linear response. Gamma-corrected or processed JPEGs cannot be used unless non-linearities are reversed [S6].
*   **Scene Extent:** Default extent calculation based on furthest camera can be biased if cameras are clustered. Use corrected extent based on SfM points [S11].
*   **Overly Large Gaussians:** Can negatively impact geometric representation, especially in background areas [S3].

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
Reduces parameter count by distilling third-order SH coefficients into first-order using a teacher-student model, preserving quality despite reducing from 16 to 4 coefficients per channel.

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field, ensuring unbiased transmittance isosurfaces and preventing surface erosion.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
Derives analytical Jacobians of SE(3) camera poses for efficient gradient-based tracking in SLAM, avoiding automatic differentiation overhead.

### DAVIGS
Mitigates floaters and color distortions by transforming rendered images using 3D consistent features queried from multi-resolution hash grids, decoupling global and local appearance variations [S1].

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses memory bandwidth and compute throughput bottlenecks by mapping software strategies to hardware constraints (Roofline model) or using comparison-free sorting.

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes 3DGS to incorporate semantic language features (e.g., CLIP) by adding a language loss, disabling densification/pruning during this phase.

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps, using dynamic texel allocation and splitting-based densification to avoid convergence issues.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
A neural field providing constant-time ray queries for secondary rays (shadows, AO) without mesh extraction or BVH traversal.

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Defines primitives for rasterization and culling, using alternative radial basis functions (Inverse Multiquadric) and opacity-independent truncation for aggressive culling.

### [DeSplat](12925-desplat.md)
Decomposes Gaussians into static and per-view distractor sets to remove transient occluders without external semantic models.

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Modifies the rasterizer for spherical rendering and uses OpenSfM to process equirectangular images from 360-degree cameras.

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint via ASCII encoding, vector clustering quantization, and hybrid hash grid/codebook representations.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Places isometric Gaussians on a fixed voxel grid for CT volume extraction and differentiable X-ray rendering, fixing positions and removing color coefficients.

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of discontinuous indicator functions for backpropagation in discontinuity-aware rendering.

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Enables rendering of sharp boundaries by pre-scissoring Gaussians along Bézier curves and using boundary gradient approximation.

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Uses second-order statistics (covariance-based feature augmentation) to reduce anchor feature dimensions while maintaining quality.

### Corrected Scene-Extent
Calculates scene extent based on SfM points rather than camera positions to avoid bias from clustered camera distributions [S11].

### SH Degree Zero Optimization
Sets SH degree to 0 to remove superfluous operations, improving training speed by ~16% [S12, unofficial].

### Selective Defocus Blurring
Predicts per-Gaussian scale offsets to create spatially varying blur kernels matching scene blurriness [S13].

### Triangle Soup (Mesh-based Gaussian Splatting)
Connects Gaussians to a mesh structure for real-time editing and animation, recalculating Gaussian parameters when vertices move [S14].

### SuGaR Mesh Extraction
Extracts surface meshes by regularizing Gaussians to align with the scene surface and applying Poisson reconstruction [S15].

### Compact View-dependent Color
Replaces per-Gaussian SH coefficients with a shared grid-based neural field, reducing parameters from 48 to a shared grid [S16].

### Denoise-GS
Combines unsupervised image denoising (Noise2Void) with 3DGS to address sparse-view and noisy scene reconstruction [S17].

## Operating characteristics

*   **Time Complexity:** Rendering is $O(N)$ per pixel where $N$ is the number of overlapping Gaussians. Training time is dominated by rasterization and gradient accumulation. Multi-view training reduces gradient variance but increases idle thread overhead if not optimized [S2, unofficial].
*   **Memory Complexity:** Scales with the number of Gaussians. Standard 3DGS consumes significant memory for large scenes. Voxel-based methods (GVKF) reduce memory by managing Gaussians in sparse grids [S3]. 3iGS uses 87.5% less voxels than TensoRF for illumination [S8].
*   **Measured Performance:** Underwater 3DGS achieves ~140 FPS inference on RTX 4090, with ~500 MB memory per scene over 30,000 steps [S6]. 3iGS achieves PSNR 27.51, SSIM 0.930 on Shiny Blender dataset [S8].
*   **Strengths:** Real-time rendering, high-fidelity novel view synthesis, efficient adaptive density control.
*   **Limitations:** High memory usage for large scenes, artifacts in transparent or specular surfaces without extensions, sensitivity to initialization and camera distribution [S1, S3, S6].
*   **When to Use:** Real-time novel view synthesis, dynamic scenes (with 4D extensions), underwater reconstruction, scenes requiring explicit surface extraction.
*   **When Not to Use:** Scenes with extreme transparency without specialized models, sparse views without denoising, non-linear camera responses.

## Background

3D Gaussian Splatting was introduced by Kerbl et al. in 2023 as a real-time explicit scene representation method [S1, S8]. It builds on the idea of splatting 3D Gaussians onto the image plane and alpha-blending them. The method has since been extended to handle various scenarios including underwater scenes [S6], dynamic 4D scenes [S5, unofficial], and surface reconstruction [S3, S15]. The implementation details described here reflect the evolution of the method to address its initial limitations in memory, appearance modeling, and robustness.

## Sources

[S1] DN-Splatter: Depth and Normal Priors for Gaussian Splatting and Meshing, https://doi.org/10.1109/wacv61041.2025.00241, fetched 2026-10-26
[S2] Efficient multi-view training for 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2506.12727, fetched 2026-10-26 (unofficial)
[S3] GVKF: Gaussian Voxel Kernel Functions for Highly Efficient Surface Reconstruction in Open Scenes, https://doi.org/10.52202/079017-3328, fetched 2026-10-26
[S4] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S5] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S6] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
[S7] A Mixed-Primitive-based Gaussian Splatting Method for Surface Reconstruction, https://core.ac.uk/works/308836756, fetched 2026-10-30 (unofficial)
[S8] 3iGS: Factorised Tensorial Illumination for 3D Gaussian Splatting, https://core.ac.uk/works/391741134, fetched 2026-10-29
[S9] Augmented Radiance Field: A General Framework for Enhanced Gaussian Splatting, https://doi.org/10.48550/arXiv.2602.19916, fetched 2026-10-30 (unofficial)
[S10] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S11] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S12] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S13] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S14] GaMeS: Mesh-based adapting and modification of Gaussian Splatting, https://doi.org/10.1016/j.patcog.2026.114561, fetched 2026-10-29
[S15] Gaussian Frosting: Editable Complex Radiance Fields with Real-Time Rendering, https://doi.org/10.1007/978-3-031-73650-6_24, fetched 2026-10-26
[S16] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S17] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
