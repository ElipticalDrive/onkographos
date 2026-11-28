# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and variant methodologies required to train and render 3D Gaussian Splatting (3DGS) scenes effectively. It addresses the gap between the theoretical formulation of differentiable rasterization and the practical engineering constraints of GPU memory, precision, and convergence stability. It sets itself apart from general training pipelines by focusing on low-level kernel behavior, specific hyperparameter configurations, and specialized extensions for handling defocus, noise, underwater scattering, and inverse rendering.

## How it works

### Standard Implementation Configuration
The baseline implementation relies on the `gsplat` library [S2] using the 3DGUT formulation [S1]. Custom CUDA kernels are added for forward and backward logic to optimize register pressure and throughput [S1].

**Data Structures and Parameters:**
*   **Optimizer:** Adam with learning rate $1 \times 10^{-3}$ [S2].
*   **Spherical Harmonics (SH):** Degree 1 is used for simplicity in standard configurations [S2].
*   **Precision:** Half-precision (FP16) memory fetches are used for feature vectors during rasterization (both forward and backward) to reduce register pressure [S1]. Neural network components (e.g., MLPs) use tiny-cuda-nn's JIT-compiled cooperative vector MLPs, trained and evaluated in FP16 [S1].
*   **Stability:** An automatic scaling factor is applied to loss values to improve half-precision training stability [S1].
*   **Iteration Strategy:** During training, a single image is randomly selected from a random scene as supervision for each iteration [S2].

**Volume Setup (Example):**
For urban scenes (e.g., Waymo/KITTI-360), an axis-aligned bounding box partitions the foreground [S2].
*   **Dimensions:** Height 12.8m, Width 32m, Length 64m [S2].
*   **Voxelization:** Voxel size 0.1m, resulting in volume dimensions $128 \times 320 \times 640$ [S2].
*   **Waymo Zero-shot Range:** Adjusted to (50m, 20m, 128m) [S2].

### Corrected Scene Extent
The default 3DGS scene extent $e_{\text{scene}}$ is proportional to the distance of the furthest camera from the average camera position $\bar{C}$ [S3]:
$$e_{\text{scene}} = 1.1 \cdot \max_{i=1}^{N_{\text{cam}}} \|\bar{C} - C_i\|_2$$
$$\bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i$$
This causes bias if cameras are clustered (e.g., orbiting a small object), leading to incorrect splitting/pruning [S3]. The corrected extent $e'_{\text{scene}}$ uses the SfM point cloud:
$$e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2$$
where $N_{\text{SfM}}$ is the number of SfM points and $p_i$ is the i-th SfM point [S3].

### Selective Defocus Blurring
To handle spatially varying blur, a compact MLP predicts a per-Gaussian scale offset $\delta s_j$ [S4].
1.  Predict $\delta s_j$ for each Gaussian $j$.
2.  Add offset to base scale to transform covariance.
3.  Project transformed covariance to 2D space.
4.  Use as pixel-wise blur kernel during rasterization [S4].
If $\delta s_j$ is constant, it degrades to uniform Gaussian blur [S4].

### Denoise-GS (Sparse-View Training)
A two-round self-supervised process combining Noise2Void (N2V) and 3DGS [S5]:
1.  **Round 1:** Train a UNet using N2V for denoising. Process denoised images through InstantSplat to initialize and train 3DGS. Render results.
2.  **Filtering:** Group and filter rendered results by viewpoint to ensure representativeness.
3.  **Round 2:** Fine-tune the UNet using a composite loss (low-frequency and perceptual detail consistency). Render again on selected viewpoints [S5].

### View-Dependent Opacity Kernel
Models specular reflectance lobes by making opacity view-dependent [S6, unofficial].
**Formula:**
$$\hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos(\max(0, \min(\frac{2\theta}{T}, \pi)))+1}{2}\exp(\beta)$$
*   $\alpha$: Base opacity.
*   $\theta$: Angle between view direction and lobe central orientation.
*   $T$: Angular span (initialized as $c\bar{\theta}_i/\pi$).
*   $\beta$: Sharpness modulator (initialized to 0).
Adds 5 learnable parameters per Gaussian [S6, unofficial]. Transparency diminishes to zero as $\theta$ increases, preventing artifacts [S6, unofficial].

### Backscatter Estimation and Optimization (Underwater)
Adapts 3DGS for underwater scenes by estimating direct attenuation $B_d$, infinite backscatter $B_\infty$, and backscatter coefficient $B_b$ [S7].
**Backscatter Estimation Algorithm:**
1.  Divide depth map (from current scene estimate) into 10 evenly-spaced clusters from $\min(z)$ to $\max(z)$ [S7].
2.  Identify pixels in ground-truth image with RGB triplets below the bottom 1st percentile within each cluster [S7].
3.  These triplets serve as an overestimate of backscatter $B_s$ [S7].
4.  Use nonlinear least squares fitting on pairs $(B_s, z)$ to estimate $B_\infty$ and $B_b$ [S7].
5.  Re-estimate parameters every 500 optimization steps [S7].
**Depth Extraction:**
$$z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i}$$
where $d_i$ is the depth of Gaussian $i$ [S7].
**Constraints:** Camera sensor must have a linear response; gamma-corrected images cannot be used unless reversed [S7]. Bounds: $B_\infty \in [0, 1]$, $B_b \in [0, 5]$ [S7].

### Multi-Scale Constraint (Large-Scale Scenes)
Applies different scales to Gaussians based on viewer distance [S8].
**Gamma Function:**
$$\gamma(\mu) = \begin{cases} 1 & \text{if } \|\mu - c\|^2 < 2r \\ \|\mu - c\|^2 / r - 1 & \text{if } \|\mu - c\|^2 \ge 2r \end{cases}$$
where $r$ is scene radius and $c$ is camera center [S8].
**Operations:**
*   Clone if $\max(s) < g \cdot \gamma(\mu)$ ($g$ is cloning threshold).
*   Split if $\max(s) \ge g \cdot \gamma(\mu)$.
*   Eliminate if $\max(s) > b \cdot \gamma(\mu)$ ($b$ is elimination threshold) [S8].

### GS-IR Representation (Inverse Rendering)
Extends vanilla 3DGS for inverse rendering by adding normal and material properties [S9].
**Layout:**
*   **Geometric ($T$):** Position $p$, scaling $s$, unit quaternion $q$, normal vector $n$ [S9].
*   **Material ($M$):** Albedo $a$, roughness $\rho$, metallicity $m$ [S9].
*   **Appearance ($A$):** Opacity $\alpha$, SH coefficients $f_c$ [S9].
**Training:**
*   **Initial Stage (30K iters):** Minimize color reconstruction loss $L_c$ and normal loss $L_n$ to optimize $T, A$.
*   **Decomposition Stage (10K iters):** Fix $T, A$; minimize decomposition loss $L_d$ to optimize $M$ [S9].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. Ensures unbiased, ellipsoidal transmittance isosurfaces, preventing surface erosion and multi-view inconsistency found in prior approximations like GGGS [9624-continuous-attenuation-model-for-3d-gaussian-splatting.md].

### DAVIGS
A plug-and-play appearance modeling method that mitigates floaters and color distortions by transforming rendered images using 3D consistent features. Decouples global view-dependent variations (e.g., daylight) from local spatial changes using multi-resolution hash grids, maintaining real-time speeds by discarding the appearance module after optimization [12021-davigs.md].

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses computational bottlenecks in 3DGS rendering by mapping software strategies to hardware constraints (Roofline model) or replacing standard sorting with hardware-friendly, comparison-free logic and sorting-free approximations [12180-sorting-and-rendering-pipeline-optimization.md].

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes 3DGS scenes to incorporate semantic language features (e.g., CLIP) alongside photometric rendering. Distinguishes itself by conditionally including language feature loss and disabling densification/pruning during language-aware training [12438-langsplat-training-procedure.md].

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussian primitives with spatially varying texture maps that offset SH colors. Uses dynamic, content-aware texel allocation and splitting-based densification to avoid convergence issues associated with cloning textured primitives [12606-content-aware-texturing-for-gaussian-splatting.md].

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Modifies the definition, spatial extent, or lifecycle of 3D Gaussian primitives. Introduces alternative kernel shapes, opacity-independent culling, voxel-based constraints, and aggressive pruning strategies to improve fidelity or efficiency [12855-gaussian-representation-and-truncation.md].

### [DeSplat](12925-desplat.md)
Decomposes 3DGS into static scene Gaussians ($G_s$) and per-view distractor Gaussians ($G_d$) to remove transient occluders (pedestrians, vehicles) without external semantic models. Relies solely on volume rendering and photometric loss for independent rendering of static and transient components [12925-desplat.md].

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Generates 3D representations from 360-degree equirectangular images by modifying the differentiable rasterizer for spherical rendering and using OpenSfM (which supports equirectangular inputs) instead of Colmap [12929-360-gaussian-splatting.md].

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces the excessive memory footprint of 3DGS by compressing numerical representations, clustering parameters into codebooks, or replacing explicit attributes with neural fields [12943-data-compression-and-storage-optimization.md].

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of a discontinuous indicator function with respect to Bézier curve control points, enabling backpropagation in discontinuity-aware rendering where standard differentiable rendering fails [13137-b-zier-boundary-gradient-approximation.md].

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Enables 3DGS to render sharp boundaries by introducing "pre-scissoring" along cubic Bézier curves and using Bézier-boundary gradient approximation to maintain differentiability [13138-discontinuity-aware-gaussian-splatting.md].

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
An anchor-based technique using covariance-based feature augmentation to capture co-variation patterns. Achieves superior quality with smaller feature vectors compared to standard anchor methods like Scaffold-GS [13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md].

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Adapts 3DGS to underwater environments by learning direct attenuation, infinite backscatter, and backscatter coefficients. Uses alternating optimization to refine depth and medium parameters simultaneously, maintaining real-time speeds [13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md].

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
A densification strategy that optimizes 2D Gaussian kernels in image space using a differentiable rasterizer, then back-projects them to 3D. Solves initialization issues for complex specular radiance where standard 3D initialization is insufficient [13985-error-driven-2d-gaussian-training.md].

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
Converts the volumetric 3DGS representation into explicit triangular meshes or point clouds. Methods include regularizing Gaussians to align with surfaces during training (SuGaR) or sampling discrete points post-training [14095-mesh-extraction-and-representation.md].

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Techniques to reduce the overhead of view-dependent appearance representation. Includes compressing SH coefficients (distillation, vector quantization), reducing polynomial degree, or optimizing SH gradient handling [14206-spherical-harmonics-optimization.md].

### [TraGraph-GS](14454-tragraph-gs.md)
Uses graph-based adaptive spatial partitioning and regularization for large-scale scenes. Decouples the scene into local regions and a coarse global representation, using a camera connectivity graph to ensure balanced subgraphs and progressive rendering to eliminate boundary artifacts [14454-tragraph-gs.md].

### [Explicit 4D Gaussian Splatting](14584-explicit-4d-gaussian-splatting.md)
Achieves memory efficiency in dynamic scenes by separating Gaussians into static and dynamic categories. Explicitly samples positions and rotations at sparse timestamps and uses interpolation, avoiding the memory overhead of multiple primitives for occluded objects [14584-explicit-4d-gaussian-splatting.md].

### [Color and Appearance Modeling](14740-color-and-appearance-modeling.md)
Extends standard 3DGS radiance representation to handle complex lighting and view-dependent effects. Decouples geometry from lighting or image-specific conditions, differing from continuous attenuation models by focusing on radiance composition [14740-color-and-appearance-modeling.md].

### [GS-IR](14852-gs-ir.md)
An inverse rendering approach using forward mapping volume rendering to estimate geometry, material, and environment illumination. Uses 3D Gaussians for real-time efficiency compared to NeRF-based backward mapping methods [14852-gs-ir.md].

## Operating characteristics

**Time and Memory Complexity:**
Standard 3DGS training is memory-intensive due to per-Gaussian attributes (position, covariance, opacity, SH coefficients). Large scenes can exceed 1 GB [12943-data-compression-and-storage-optimization.md]. Implementations like `gsplat` use half-precision (FP16) to reduce register pressure and memory bandwidth usage [S1].

**Performance:**
*   **Hardware:** Training and evaluation often occur on high-end GPUs (e.g., NVIDIA V100, RTX4090) [S2][S9].
*   **Speed:** 3DGS offers real-time rendering speeds (e.g., 81-83 FPS on KITTI-360/Waymo datasets) compared to slower NeRF-based methods [S2].
*   **Accuracy:** Methods like EVolSplat achieve PSNR of 23.26 dB (KITTI-360) and 23.43 dB (Waymo) with low memory usage (10.41 GB) [S2].

**Strengths and Limitations:**
*   **Strengths:** Real-time rendering, high fidelity, explicit representation allows for easy editing and mesh extraction.
*   **Limitations:** High memory footprint, sensitivity to camera distribution (addressed by Corrected Scene Extent), struggles with specular highlights (addressed by View-Dependent Opacity), and poor performance in sparse/noisy views (addressed by Denoise-GS) or underwater environments (addressed by Backscatter Estimation) [S3][S4][S5][S6, unofficial][S7].

## Background

3D Gaussian Splatting was introduced as a method for real-time radiance field rendering using explicit 3D Gaussian primitives. It has since evolved through various optimizations and extensions to handle specific challenges such as large-scale scenes, inverse rendering, and non-standard environments. The implementation details described here stem from various research efforts aiming to improve stability, accuracy, and efficiency of the core 3DGS framework.

## Sources

[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S5] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S6] Augmented Radiance Field: A General Framework for Enhanced Gaussian Splatting, https://doi.org/10.48550/arXiv.2602.19916, fetched 2026-10-30 (unofficial)
[S7] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
[S8] TraGraph-GS: Trajectory Graph-based Gaussian Splatting for Arbitrary Large-Scale Scene Rendering., https://doi.org/10.1109/tpami.2026.3709368, fetched 2026-10-26
[S9] GS-IR: 3D Gaussian Splatting for Inverse Rendering, https://doi.org/10.1109/cvpr52733.2024.02045, fetched 2026-10-26
