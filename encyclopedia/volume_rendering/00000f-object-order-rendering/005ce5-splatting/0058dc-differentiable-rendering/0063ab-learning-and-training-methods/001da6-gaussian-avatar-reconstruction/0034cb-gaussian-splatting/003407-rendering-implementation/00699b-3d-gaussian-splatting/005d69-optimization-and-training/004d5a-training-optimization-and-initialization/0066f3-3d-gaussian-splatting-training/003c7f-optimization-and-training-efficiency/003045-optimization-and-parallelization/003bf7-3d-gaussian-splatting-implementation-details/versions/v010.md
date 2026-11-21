# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and architectural modifications required to train and render 3D Gaussian Splatting (3DGS) efficiently. It addresses the gap between the theoretical formulation of differentiable volume rendering and practical software engineering, focusing on memory management, precision handling, and hyperparameter tuning. It sets itself apart from general training pipeline descriptions by providing concrete values for optimizer settings, data structures, and kernel-level optimizations (e.g., half-precision fetches) that are critical for achieving real-time performance and stable convergence.

## How it works
The implementation relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters.

### Inputs and Outputs
*   **Inputs:** A set of $N$ 3D Gaussians, each defined by mean position $\mu \in \mathbb{R}^3$, covariance matrix $\Sigma \in \mathbb{R}^{3 \times 3}$ (or rotation $R \in SO(3)$ and scale $s \in \mathbb{R}^3$), opacity $\alpha \in [0, 1]$, spherical harmonic coefficients $c_{l,m}$, and view direction $d$. Camera poses $C_i$ and intrinsics $K$ for $V$ views.
*   **Outputs:** Rendered images $\hat{I} \in \mathbb{R}^{H \times W \times 3}$ and gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ for all parameters $\theta$.

### Data Structures
1.  **Gaussian List:** A flat array of $N$ Gaussians. Each Gaussian stores:
    *   `pos`: `float3` (32-bit float, 12 bytes).
    *   `rot`: `float4` (quaternion, 16 bytes).
    *   `scale`: `float3` (24 bytes).
    *   `opacity`: `float` (4 bytes).
    *   `sh_dc`: `float3` (12 bytes, 0-degree SH).
    *   `sh_rest`: `float` array (variable size, e.g., 45 floats for degree 3, 180 bytes).
    *   *Note:* In optimized implementations (e.g., `gsplat`), feature vectors may be fetched in half-precision (FP16) to reduce register pressure [S1].
2.  **Tile/Screen Space Data:** For each tile (e.g., $16 \times 16$ pixels), a list of overlapping Gaussians sorted by depth.
3.  **Gradients:** Accumulators for $\frac{\partial \mathcal{L}}{\partial \mu}, \frac{\partial \mathcal{L}}{\partial R}, \frac{\partial \mathcal{L}}{\partial s}, \frac{\partial \mathcal{L}}{\partial \alpha}, \frac{\partial \mathcal{L}}{\partial c}$.

### Training Steps
1.  **Initialization:** Initialize Gaussians from Structure-from-Motion (SfM) points. Calculate scene extent $e_{\text{scene}}$.
    *   *Default:* $e_{\text{scene}} = 1.1 \cdot \max_{i} \|\bar{C} - C_i\|_2$ [S3].
    *   *Corrected:* $e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2$ to avoid bias from clustered cameras [S3].
2.  **Forward Pass (Rasterization):**
    *   Project 3D Gaussians to 2D screen space using camera matrices.
    *   Compute 2D covariance $\Sigma_{2D}$ and screen-space radius.
    *   Sort Gaussians by depth for each pixel/tile.
    *   Alpha-blend colors: $C_{out} = \sum_i c_i \alpha_i T_i$, where $T_i = \prod_{j<i} (1-\alpha_j)$ [Parent].
    *   *Optimization:* Use half-precision memory fetches for feature vectors in CUDA kernels [S1].
3.  **Loss Computation:**
    *   Compute $\mathcal{L} = \mathcal{L}_{color} + \lambda_{ssim} \mathcal{L}_{SSIM}$.
    *   *Half-Precision Stability:* Apply an automatic scaling factor to loss values to improve FP16 training stability [S1].
4.  **Backward Pass:**
    *   Compute gradients via backpropagation through the rasterizer.
    *   *Optimization:* Use tiny-cuda-nn's JIT-compiled cooperative vector MLPs for neural appearance components, trained in FP16 [S1].
5.  **Optimization Step:**
    *   Update parameters using Adam optimizer.
    *   *Standard LR:* $1 \times 10^{-3}$ [S2].
    *   *Densification:* Split/clone Gaussians based on gradient thresholds.
    *   *Pruning:* Remove low-opacity Gaussians.

### Parameters and Configuration
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** $1 \times 10^{-3}$ [S2].
*   **SH Degree:** Default 3, but can be set to 0 for speed [S5, unofficial].
*   **Background Color:** Default white (1.0, 1.0, 1.0). Optimal range for speed/quality balance is $0.3 \le bg \le 0.5$ [S5, unofficial]. Setting $bg=0.4$ reduces processing time by ~21.85% on some datasets [S5, unofficial].
*   **Volume Dimensions (Example):** Height 12.8m, Width 32m, Length 64m. Voxel size 0.1m, resulting in $128 \times 320 \times 640$ volume [S2].
*   **Waymo Zero-shot Volume Range:** (50m, 20m, 128m) [S2].

### Edge Cases and Pitfalls
*   **Scene Extent Bias:** Default extent calculation based on camera positions can be too small for large outdoor scenes if cameras are clustered, leading to incorrect splitting/pruning [S3]. Use SfM-based extent correction [S3].
*   **SH Degree > 0 Unchanged:** In standard 3DGS, SH coefficients with degree > 0 are often unchanged during training, making them superfluous. Setting max degree to 0 improves training speed by ~16.25-16.57% without quality loss [S5, unofficial].
*   **Half-Precision Instability:** FP16 training can suffer from overflow/underflow. Automatic loss scaling is required [S1].
*   **Large Mesh Faces:** In mesh-based variants, large faces cause artifacts when edited. Subdivision is required [S7].

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
Reduces parameter count by distilling high-fidelity third-order SH coefficients into a compact first-order representation using a teacher-student model, preserving quality despite reducing from 16 to 4 coefficients per channel.

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$, ensuring unbiased transmittance isosurfaces and preventing surface erosion.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
Derives analytical Jacobians of SE(3) camera poses with respect to Gaussians for efficient SLAM tracking, avoiding the overhead of automatic differentiation during rasterization.

### DAVIGS
A plug-and-play method that mitigates floaters and color distortions by transforming rendered images using 3D consistent features queried from multi-resolution hash grids, decoupling global view-dependent variations from local changes [S1].

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Aligns software strategies with hardware bottlenecks (identified via Roofline model), applying memory-centric optimizations to sorting and compute-centric optimizations to alpha blending.

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes 3DGS scenes to incorporate semantic language features (e.g., CLIP) alongside photometric rendering, disabling densification/pruning during language-aware training.

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps and dynamic texel allocation, avoiding convergence issues associated with cloning textured primitives.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
A neural field providing constant-time, constant-memory ray queries for secondary rays (shadows, GI) without mesh extraction or BVH traversal.

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Defines primitives for rasterization and culling, offering alternative radial basis functions (Inverse Multiquadric) and opacity-independent truncation for aggressive culling.

### [Atomized Proliferation](12924-atomized-proliferation.md)
Constrains fine-detail Gaussians into uniform "Atom Gaussians" and uses a progressive split schedule to align densification with scene geometry, stabilizing the Gaussian count.

### [DeSplat](12925-desplat.md)
Decomposes the scene into static scene Gaussians and per-view distractor Gaussians to remove transient occluders (pedestrians, vehicles) without external semantic models.

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Modifies the rasterizer for spherical rendering and uses OpenSfM to process equirectangular images from 360-degree cameras, bypassing the need for cropping.

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint via ASCII-based encoding, sensitivity-aware vector clustering for quantization, and hybrid hash-grid/codebook representations.

### Corrected Scene-Extent
Calculates scene extent based on the SfM point cloud ($e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum \|\bar{C} - p_i\|_2$) rather than camera positions to avoid bias from clustered camera captures [S3].

### Optimization Strategy
Optimizes static Gaussians for 3000 iterations as a warm-up, then jointly optimizes Gaussians and deformation fields. Loss is $L = L_{color} + \lambda_1 L_{scales}$ with $\lambda_1 = 0.1$ [S4, unofficial].

### SH Degree Zero Optimization
Sets max SH degree to 0 and removes superfluous operations, improving training speed by ~16.57% as higher-degree coefficients remain unchanged during standard training [S5, unofficial].

### Background Color Optimization
Changes background from white to gray (optimal $0.3 \le bg \le 0.5$) to improve training speed (e.g., 21.85% reduction in processing time) while maintaining quality [S5, unofficial].

### Selective Defocus Blurring
Predicts per-Gaussian scale offsets $\delta s_j$ via a compact MLP to create pixel-wise variable blur kernels, modeling spatially varying scene blurriness more effectively than uniform Gaussian blur [S6].

### Triangle Soup (Mesh-based Gaussian Splatting)
Connects Gaussians to a mesh structure (Triangle Soup) for real-time editing. Requires subdivision of large faces to avoid artifacts and automatic recalculation of Gaussian parameters when vertices move [S7].

### SuGaR Mesh Extraction
Extracts a surface mesh by regularizing Gaussians to align with the scene surface and applying Poisson reconstruction on points sampled from the visible level set. Automatically tunes octree depth $D$ to reduce artifacts [S8].

### Compact View-dependent Color
Replaces per-Gaussian SH coefficients (48 params) with a shared grid-based neural field. Positions are contracted to a bounded range using $contract(p_n) = p_n$ if $\|p_n\| \le 1$, else $\frac{p_n}{\|p_n\|}(2-\|p_n\|)$, and fed into a tiny MLP [S9].

### AtomGS
Combines Atomized Proliferation (uniform Atom Gaussians) with Geometry-Guided Optimization (Edge-Aware Normal Loss) to smooth flat surfaces while preserving details, and merges large Gaussians during pruning [S10, unofficial].

## Operating characteristics
*   **Time Complexity:** Training is dominated by the rasterization and gradient computation, which scales with the number of Gaussians $N$ and pixels $P$. Sorting adds $O(N \log N)$ per tile.
*   **Memory Complexity:** $O(N \cdot D)$ where $D$ is the dimensionality of Gaussian attributes (e.g., 48 floats for SH degree 3). Large scenes require significant VRAM.
*   **Performance:** Real-time rendering (100+ FPS) is achievable on modern GPUs using optimized CUDA kernels (e.g., `gsplat`) [S1, S2].
*   **Strengths:** High visual fidelity, real-time rendering, explicit representation.
*   **Limitations:** High memory usage for large scenes, potential for "floaters" in unobserved regions, sensitivity to hyperparameters (e.g., scene extent, background color) [S3][S5, unofficial].
*   **When to Use:** Real-time novel view synthesis, interactive applications.
*   **When Not to Use:** Scenes with extreme transient occlusions (use DeSplat) or where mesh extraction is primary goal (use SuGaR/Triangle Soup) [S1, S7, S8].

## Background
3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1, S7, S8, S9][S10, unofficial]. It builds on the idea of representing radiance fields with explicit 3D Gaussians, allowing for fast differentiable rasterization. The method has since been extended with various optimization strategies, compression techniques, and specialized variants for dynamic scenes, mesh extraction, and semantic understanding.

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Alias-free 4D Gaussian Splatting, https://arxiv.org/abs/2511.18367v1, fetched 2026-10-26 (unofficial)
[S5] I3DGS: Improve 3D Gaussian Splatting from Multiple Dimensions, https://arxiv.org/abs/2405.06408v1, fetched 2026-10-26 (unofficial)
[S6] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S7] GaMeS: Mesh-based adapting and modification of Gaussian Splatting, https://doi.org/10.1016/j.patcog.2026.114561, fetched 2026-10-29
[S8] Gaussian Frosting: Editable Complex Radiance Fields with Real-Time Rendering, https://doi.org/10.1007/978-3-031-73650-6_24, fetched 2026-10-26
[S9] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S10] AtomGS: Atomizing Gaussian Splatting for High-Fidelity Radiance Field, https://core.ac.uk/works/391753628, fetched 2026-10-30 (unofficial)
