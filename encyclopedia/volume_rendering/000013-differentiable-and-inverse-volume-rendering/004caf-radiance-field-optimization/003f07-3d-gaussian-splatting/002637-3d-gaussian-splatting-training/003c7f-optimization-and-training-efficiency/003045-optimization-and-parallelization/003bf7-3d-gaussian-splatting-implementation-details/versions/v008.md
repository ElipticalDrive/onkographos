# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and algorithmic variants required to train and render 3D Gaussian Splatting (3DGS) scenes efficiently. It addresses the gap between the theoretical differentiable rasterization framework and practical execution, covering memory management, densification logic, and hardware-specific optimizations. It distinguishes itself from general training pipelines by focusing on concrete hyperparameters, data structure layouts, and kernel-level modifications (e.g., half-precision, spherical rendering) necessary for stable convergence and performance.

## How it works

### Inputs and Outputs
*   **Inputs:** Sparse point cloud (positions, covariances), camera poses (extrinsics, intrinsics), input images, and hyperparameters (learning rates, thresholds).
*   **Outputs:** Optimized Gaussian attributes (mean $\mu$, covariance $\Sigma$, opacity $\alpha$, spherical harmonic coefficients $c_l^m$, or grid features), rendered images, and loss gradients.

### Data Structures
*   **Gaussian Attributes:** Stored in contiguous arrays for GPU access.
    *   **Position:** `float3` (x, y, z).
    *   **Covariance:** Represented as a 3D rotation matrix $R$ (derived from quaternion) and scale vector $s$. In memory, often stored as quaternion $(w, x, y, z)$ and scale $(s_x, s_y, s_z)$ [S4, unofficial].
    *   **Opacity:** `float` (alpha).
    *   **Color:**
        *   Standard 3DGS: Spherical Harmonic (SH) coefficients. Degree 0: 3 floats (RGB). Degree 1: 12 floats. Degree 2: 27 floats. Degree 3: 48 floats per Gaussian [S1].
        *   Compact 3DGS: Codebook indices for geometry and grid features for color [S1].
*   **Camera Parameters:** Intrinsics (focal length, principal point) and Extrinsics (4x4 transformation matrix) [S3, unofficial].
*   **Scene Extent:** Used for normalization and splitting thresholds. Default: $e_{\text{scene}} = 1.1 \cdot \max_{i} \|\bar{C} - C_i\|_2$ [S6].

### Optimization and Training Steps

#### 1. Initialization
*   **Point Cloud Generation:** Use Structure-from-Motion (SfM) tools.
    *   Standard: COLMAP (perspective cameras) [S3, unofficial].
    *   Equirectangular: OpenSfM (supports 360° images directly) [S3, unofficial].
    *   Read positions from `reconstruction.json`. Ignore `error` values (set to 0) [S3, unofficial].
*   **Initial Covariance:** Set initial scale based on the average distance to nearest neighbors in the SfM point cloud [S4, unofficial].
*   **Scene Extent Calculation:**
    *   Default: Based on furthest camera [S6].
    *   Corrected: $e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2$, where $\bar{C}$ is the average camera position and $p_i$ are SfM points [S6]. This prevents bias when cameras are clustered [S6].

#### 2. Forward Pass (Rasterization)
*   **Projection:** Project 3D Gaussians to 2D screen space using the camera matrix.
*   **Sorting:** Sort Gaussians by depth for each pixel or tile to ensure correct alpha blending [Parent].
*   **Rendering:**
    *   Standard: Perspective projection [S3, unofficial].
    *   Spherical/Equirectangular: Modify the rasterization kernel (`auxiliary.h`) to map 3D points to spherical coordinates $(\lambda, \phi)$ [S3, unofficial]:
        ```python
        # Pseudocode for spherical projection in CUDA
        direction = transform_point(p, view_matrix)
        longitude = atan2(direction.x, direction.z)
        latitude = atan2(direction.y, sqrt(direction.x^2 + direction.z^2))
        normalized_lon = longitude / pi
        normalized_lat = latitude / (pi / 2)
        ```
*   **Alpha Blending:** Composite colors using transmittance $T_i$ and opacity $\alpha_i$ [Parent].

#### 3. Backward Pass (Gradient Computation)
*   Compute gradients of the loss $\mathcal{L}$ (e.g., L1 + SSIM) with respect to Gaussian parameters [Parent].
*   **Half-Precision Optimization:** Use FP16 memory fetches for feature vectors in forward and backward kernels to reduce register pressure [S2]. Apply automatic scaling to loss values to stabilize FP16 training [S2].

#### 4. Densification and Pruning (AtomGS Strategy)
The standard 3DGS densification is replaced or augmented by the Atomized Proliferation algorithm [S4, unofficial]:

1.  **Prune:** If opacity $\alpha < \epsilon$ OR covariance norm $\|\Sigma\|_{\text{max}}$ is too large, remove the Gaussian.
    *   $\epsilon = 0.005$ (standard), $\epsilon = 0.5$ (DTU dataset) [S4, unofficial].
2.  **Clone:** If loss gradient $\nabla_p \mathcal{L} \ge \tau_c$, clone the Gaussian.
    *   $\tau_c = 0.002$ [S4, unofficial].
3.  **Split:** If $\nabla_p \mathcal{L} \ge \min(\frac{i}{t_w}\tau_s, \tau_s)$ AND $\|\Sigma\|_{\text{max}} > S$, split the Gaussian.
    *   $\tau_s = 0.002$ [S4, unofficial].
    *   $t_w = 7000$ (warm-up duration) [S4, unofficial].
    *   Linear warm-up increases the split threshold over time [S4, unofficial].
4.  **Atomize:** If $\|\Sigma\|_{\text{min}} \le S$ AND iteration $i < t_a$, convert to an Atom Gaussian.
    *   $S = P_1(d)$ (1st percentile of SfM neighbor distances) [S4, unofficial].
    *   $t_a = 7000$ (atomization duration) [S4, unofficial].
    *   Atom Gaussians are isotropic spheroids with fixed scale $s = \sqrt[3]{S}$, updated via geometric progression $S_n = S_{n-1} \cdot r^{n-1}$ where $r = \sqrt[t_a]{p}$ [S4, unofficial].

#### 5. Geometry-Guided Optimization
*   Compute Edge-Aware Normal Loss by integrating 2D edge detector weights into the curvature map [S4, unofficial].
*   Loss: $\mathcal{L} = \mathcal{L}_{\text{color}} + \lambda_1 \mathcal{L}_{\text{scales}} + \lambda_{\text{normal}} \mathcal{L}_{\text{normal}}$ [S4, unofficial].
*   $\lambda_1 = 0.1$, $\lambda_{\text{normal}} = 0.1$ [S4, unofficial].

### Parameters
*   **Optimizer:** Adam, learning rate $1 \times 10^{-3}$ [S5].
*   **SH Degree:** 1 (standard), 0 (for speed optimization) [S5][S8, unofficial].
*   **Background Color:** Default white (1.0). Optimized gray value $bg \in [0.3, 0.5]$ improves speed by ~21.85% on truck dataset [S8, unofficial].
*   **Volume Dimensions:** Example: $128 \times 320 \times 640$ voxels (0.1m size) for Waymo dataset [S5].

### Edge Cases and Pitfalls
*   **Scene Extent Bias:** Default extent calculation fails if cameras orbit a small object in a large scene, causing incorrect splitting/pruning. Use SfM-based extent [S6].
*   **SH Degree > 0:** Coefficients for degree > 0 are often unchanged during training, adding computational overhead without benefit. Set max degree to 0 for 16.25% speedup [S8, unofficial].
*   **Half-Precision Instability:** Requires loss scaling and careful handling of gradient magnitudes [S2].
*   **Equirectangular Rendering:** Standard rasterizers assume perspective. Must modify projection kernels [S3, unofficial].

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
Reduces parameter count by distilling high-fidelity third-order SH coefficients into a compact first-order representation using a teacher-student framework, preserving quality despite reducing coefficients from 16 to 4 per channel.

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field, ensuring unbiased transmittance isosurfaces and preventing surface erosion.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
Derives analytical Jacobians of SE(3) camera poses for efficient gradient-based tracking in SLAM, avoiding the overhead of automatic differentiation during rasterization.

### [ASCII-Based Data Compression](11820-ascii-based-data-compression.md)
Compresses floating-point and integer values into ASCII strings to accelerate data transmission during training, achieving ~6% speed improvement.

### DAVIGS
Mitigates appearance variation-induced floaters by transforming rendered images using 3D consistent features queried from multi-resolution hash grids, decoupling global view-dependent variations from local changes [S1].

### [Sensitivity-Aware Vector Clustering](12167-sensitivity-aware-vector-clustering.md)
Compresses SH coefficients and shape features into codebooks using sensitivity-aware vector quantization, prioritizing parameters that significantly affect image quality.

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Aligns software strategies with hardware bottlenecks (Roofline model), applying memory-centric optimizations to sorting and compute-centric optimizations to alpha blending.

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes scenes to incorporate semantic language features (e.g., CLIP) alongside photometric rendering, disabling densification/pruning during language-aware training.

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps, using dynamic texel allocation and splitting-based densification to manage appearance errors.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
A neural field providing constant-time ray queries for secondary rays (shadows, AO) without mesh extraction or BVH traversal.

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Defines primitives for rasterization, offering alternative radial basis functions (Inverse Multiquadric) and opacity-independent truncation for aggressive culling.

### [7D Gaussian Splatting](12903-7d-gaussian-splatting.md)
Represents dynamic scenes with view-dependent effects as seven-dimensional Gaussians (3D space, 1D time, 3D viewing direction).

### Corrected Scene-Extent
Calculates scene extent based on the SfM point cloud ($e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum \|\bar{C} - p_i\|_2$) rather than camera positions to avoid bias from clustered camera distributions [S6].

### Optimization Strategy
Optimizes static Gaussians for 3000 iterations as a warm-up, then jointly optimizes Gaussians and deformation fields. Uses loss $\mathcal{L} = \mathcal{L}_{\text{color}} + 0.1 \mathcal{L}_{\text{scales}}$ [S7, unofficial].

### SH Degree Zero Optimization
Sets max SH degree to 0 to remove superfluous operations, yielding a 16.25% speed improvement [S8, unofficial].

### Background Color Optimization
Changes background from white to gray (0.3-0.5) to improve training speed by ~21.85% without significant quality loss [S8, unofficial].

### Selective Defocus Blurring
Predicts per-Gaussian scale offsets via a compact MLP to create spatially varying blur kernels, matching scene blurriness [S9].

### Triangle Soup (Mesh-based Gaussian Splatting)
Connects Gaussians to a mesh structure for real-time editing. Uses 100 Gaussians per face and handles large faces via subdivision to avoid artifacts [S10].

### SuGaR Mesh Extraction
Extracts meshes by regularizing Gaussians to align with the scene surface and applying Poisson reconstruction on sampled points from the visible level set [S11].

### Compact View-dependent Color
Replaces per-Gaussian SH coefficients with a grid-based neural field (hash grids + tiny MLP) and contraction function, reducing parameters from 48 to a shared grid [S1].

## Operating characteristics

*   **Time Complexity:** Linear with respect to the number of Gaussians and pixels. Densification adds overhead proportional to the number of active Gaussians [S1].
*   **Memory Complexity:**
    *   Standard 3DGS: High, due to per-Gaussian SH coefficients (up to 48 floats) [S1].
    *   Compact 3DGS: Reduced by ~15-28x using grid-based color and codebook geometry [S1].
    *   VRAM Usage: Training images loaded only during training; ~11GB VRAM for 5K images on 16GB GPU [S3, unofficial].
*   **Performance:**
    *   Compact 3DGS: ~40% faster rendering, >25x compression with post-processing [S1].
    *   SH Degree 0: 16.25% faster training [S8, unofficial].
    *   Background Optimization: 21.85% faster training on truck dataset [S8, unofficial].
*   **Strengths:** Real-time rendering, high fidelity, efficient memory with compression techniques.
*   **Limitations:** High memory usage in standard configuration, sensitivity to initialization (SfM quality), potential artifacts with dynamic scenes without 4D extensions.

## Background

3D Gaussian Splatting was introduced by Kerbl et al. (2023) as a real-time radiance field representation. It uses 3D Gaussians as primitives for differentiable rasterization. Subsequent work has focused on optimizing training efficiency, memory usage, and geometric accuracy. Key developments include compact representations using neural fields and codebooks [S1], atomized proliferation for geometric precision [S4, unofficial], and support for equirectangular images [S3, unofficial].

## Sources

[S1] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S3] 360度カメラ画像からの空間再構成 - 360 Gaussian Splatting, https://qiita.com/inuex35/items/af720689e253cd3b617b, fetched 2026-10-29 (unofficial)
[S4] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
[S5] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S6] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S7] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
[S8] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S9] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S10] GaMeS: Mesh-based adapting and modification of Gaussian Splatting, https://doi.org/10.1016/j.patcog.2026.114561, fetched 2026-10-29
[S11] Gaussian Frosting: Editable Complex Radiance Fields with Real-Time Rendering, https://doi.org/10.1007/978-3-031-73650-6_24, fetched 2026-10-26
