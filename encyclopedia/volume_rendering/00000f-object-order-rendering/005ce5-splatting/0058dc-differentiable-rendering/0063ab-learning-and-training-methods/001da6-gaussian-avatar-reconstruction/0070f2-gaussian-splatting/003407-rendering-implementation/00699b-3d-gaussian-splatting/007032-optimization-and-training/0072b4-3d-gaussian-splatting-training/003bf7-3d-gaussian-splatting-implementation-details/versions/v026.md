# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and variant algorithms required to train and render 3D Gaussian Splatting (3DGS) scenes effectively. It addresses the gap between the theoretical differentiable rasterization framework and practical execution, covering memory management, precision handling, and specific architectural modifications for challenging environments (e.g., underwater, sparse views, large-scale scenes). It builds on [Optimization and Parallelization](6233-optimization-and-parallelization.md) by providing the concrete hyperparameters and kernel-level optimizations that enable the shared principles to function efficiently.

## How it works

### Standard Training Configuration
The baseline implementation relies on the `gsplat` library [S2] using the 3DGUT formulation [S1]. The training loop optimizes Gaussian parameters (position, covariance, opacity, color, SH coefficients) using the Adam optimizer with a learning rate of $1 \times 10^{-3}$ [S2]. Supervision is provided by randomly selecting a single image from a random scene for each iteration [S2].

### Precision and Memory Optimization
To reduce register pressure and memory bandwidth usage, the implementation employs half-precision (FP16) arithmetic throughout the rasterization pipeline.
1.  **Feature Fetches:** Half-precision memory fetches are used for feature vectors during both forward and backward rasterization kernels [S1].
2.  **Neural Networks:** Any auxiliary neural networks (e.g., for view-dependent color or appearance) are implemented using tiny-cuda-nn's JIT-compiled cooperative vector MLPs, trained and evaluated in FP16 [S1].
3.  **Loss Scaling:** An automatic scaling factor is applied to loss values to improve training stability in half-precision [S1].

### Scene Extent Calculation
The default 3DGS scene extent $e_{\text{scene}}$ is calculated based on camera positions:
$$ e_{\text{scene}} = 1.1 \cdot \max_{i=1}^{N_{\text{cam}}} \|\bar{C} - C_i\|_2 $$
where $\bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i$ is the average camera position [S3]. This method is biased if cameras are clustered (e.g., orbiting a small object in a large environment), leading to incorrect densification [S3].

**Corrected Scene Extent:** To mitigate this, the scene extent can be calculated based on the Structure-from-Motion (SfM) point cloud:
$$ e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2 $$
where $N_{\text{SfM}}$ is the number of SfM points and $p_i$ is the $i$-th SfM point [S3].

### Variant: Selective Defocus Blurring
To handle spatially varying blur, a per-Gaussian scale offset $\delta s_j$ is predicted using a compact MLP [S4].
1.  Predict $\delta s_j$ for each Gaussian $j$.
2.  Add $\delta s_j$ to the base scale to transform the covariance matrix.
3.  Project the transformed covariance to 2D space to act as a pixel-wise blur kernel during rasterization [S4].
*Pitfall:* If $\delta s_j$ is constant for all Gaussians, the method degrades to a uniform Gaussian blur [S4].

### Variant: Compact View-dependent Color
To reduce the parameter count from 48 (SH degree 3) per Gaussian, a grid-based neural field is used [S5].
1.  Contract unbounded positions $p_n \in \mathbb{R}^3$ to a bounded range:
    $$ \text{contract}(p_n) = \begin{cases} p_n & \text{if } \|p_n\| \le 1 \\ \frac{p_n}{\|p_n\|} (2 - \|p_n\|) & \text{if } \|p_n\| > 1 \end{cases} $$
2.  Input contracted positions into hash grids.
3.  Feed the resulting features and view direction $d$ into a tiny MLP $f(\cdot; \theta)$ to compute color:
    $$ c_n(d) = f(\text{contract}(p_n), d; \theta) $$
4.  Use 0-degree SH components (RGB channels) as input features [S5].

### Variant: Denoise-GS
For sparse-view and noisy scenes, a two-round self-supervised denoising process is applied [S6]:
1.  **Round 1:** Train a UNet using Noise2Void (N2V) for unsupervised denoising. Process denoised images through InstantSplat to initialize and train 3DGS.
2.  **Filtering:** Group and filter rendered results by viewpoint to ensure representativeness.
3.  **Round 2:** Fine-tune the UNet using a composite loss (low-frequency and perceptual detail consistency). Render again on selected viewpoints [S6].

### Variant: View-Dependent Opacity Kernel
To model specular reflectance, opacity $\hat{\alpha}$ is made view-dependent [S7, unofficial]:
$$ \hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos(\max(0, \min(\frac{2\theta}{T}, \pi)))+1}{2}\exp(\beta) $$
where $\alpha$ is base opacity, $\theta$ is the angle between view direction and lobe orientation, $T$ controls angular span, and $\beta$ modulates sharpness [S7, unofficial].
*Parameters:* $T$ is initialized as $c\bar{\theta}_i/\pi$; $\beta$ is initialized to zero [S7, unofficial].
*Pitfall:* As $\theta$ increases, transparency diminishes to zero, preventing artifacts outside the lobe [S7, unofficial].

### Variant: Underwater Rendering (Gaussian Splashing)
For underwater scenes, direct attenuation $B_d$, infinite backscatter $B_\infty$, and backscatter coefficient $B_b$ are learned [S8].
1.  **Backscatter Estimation:**
    *   Divide depth map into 10 clusters.
    *   Identify pixels with RGB triplets in the bottom 1st percentile as overestimates of backscatter $B_s$.
    *   Fit $B_\infty$ and $B_b$ using nonlinear least squares on $(B_s, z)$ pairs.
    *   Re-estimate every 500 steps [S8].
2.  **Depth Extraction:** Extract depth $z$ from the current scene estimate for parameter updates:
    $$ z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i} $$
    where $d_i$ is the depth of Gaussian $i$ [S8].
*Pitfall:* Camera sensor must have a linear response; gamma-corrected images cannot be used [S8].
*Bounds:* $B_\infty \in [0, 1]$, $B_b \in [0, 5]$ [S8].

### Variant: Multi-Scale Constraint
For large-scale scenes, a position-aware adaptive control adjusts Gaussian scales based on viewer distance [S9]:
$$ \gamma(\mu) = \begin{cases} 1 & \text{if } \|\mu - c\|^2 < 2r \\ \|\mu - c\|^2 / r - 1 & \text{if } \|\mu - c\|^2 \ge 2r \end{cases} $$
where $r$ is scene radius and $c$ is camera center [S9].
*   **Clone:** If $\max(s) < g \cdot \gamma(\mu)$, clone the Gaussian.
*   **Split:** If $\max(s) \ge g \cdot \gamma(\mu)$, split the Gaussian.
*   **Eliminate:** If $\max(s) > b \cdot \gamma(\mu)$, eliminate the Gaussian.
Here $g$ and $b$ are scale threshold parameters [S9].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. Ensures unbiased, ellipsoidal transmittance isosurfaces and prevents surface erosion. Choose when geometric fidelity and multi-view consistency are critical.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
Derives analytical Jacobians of SE(3) camera poses with respect to Gaussians for efficient SLAM. Solves the overhead of automatic differentiation. Choose for real-time tracking applications.

### [DAVIGS](12021-davigs.md)
Mitigates floaters and color distortions by transforming rendered images using 3D consistent features queried from hash grids. Decouples global appearance variations from local changes. Choose for scenes with significant lighting or ISP variations.

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses memory bandwidth and compute throughput bottlenecks. Maps software strategies to hardware constraints (Roofline model) or uses sorting-free approximations. Choose for maximizing rendering FPS on constrained hardware.

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes scenes to incorporate semantic language features (e.g., CLIP) alongside photometric rendering. Disables densification/pruning during language-aware training. Choose for semantic query support.

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps. Uses dynamic texel allocation and splitting-based densification. Choose for high-fidelity texture reconstruction without pretrained models.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
A neural field mapping rays to distance and visibility for constant-time ray queries (shadows, GI). Eliminates mesh extraction or BVH traversal. Choose for secondary ray effects in Gaussian scenes.

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Modifies spatial extent, density, and survival criteria using algebraic decay and geometry-guided proliferation. Focuses on the representation itself rather than rendering throughput. Choose for improved geometric fidelity and reduced redundancy.

### [DeSplat](12925-desplat.md)
Decomposes Gaussians into static ($G_s$) and per-view distractor ($G_d$) sets to remove transient occluders. Uses only volume rendering and photometric loss. Choose for distractor-free novel view synthesis.

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Adapts 3DGS for 360-degree equirectangular images using OpenSfM and spherical rasterization. Choose for 360-degree camera inputs.

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint via compression, clustering, or neural fields. Focuses on data density rather than rendering speed. Choose for large scenes with storage constraints.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Places isometric Gaussians on a fixed voxel grid for CT volume extraction and X-ray rendering. Fixes positions and removes color coefficients. Choose for medical imaging or X-ray applications.

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of discontinuous indicator functions for Bézier curves. Enables backpropagation in discontinuity-aware rendering. Choose for sharp boundary representation.

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Introduces "pre-scissoring" along Bézier boundaries to handle sharp discontinuities. Choose for scenes with hard edges or occlusion boundaries.

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Uses second-order statistics and covariance-based feature augmentation to reduce anchor feature dimensions. Choose for high-quality rendering with smaller model size.

### [3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md)
Models outgoing radiance as a function of local illumination and BRDF attributes. Accounts for inter-Gaussian illumination. Choose for accurate specular effects on reflective surfaces.

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Replaces discrete opacity with Gaussian kernel functions for surface reconstruction. Enables direct mesh extraction. Choose for high-fidelity open scene surface reconstruction.

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Integrates ocean optics models (attenuation, backscatter) for underwater scenes. Choose for aquatic environments with color distortion.

### [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md)
Enables multi-view mini-batch training to reduce gradient variance. Uses thread-efficient partial rendering. Choose for stable optimization with multiple views.

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Optimizes 2D Gaussians in image space and back-projects to 3D. Solves initialization for complex specular radiance. Choose for view-dependent specular reconstruction.

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Reduces SH computational cost via distillation, quantization, or degree reduction. Choose for memory-constrained appearance representation.

### [SWAG (Splatting in the Wild images with Appearance-conditioned Gaussians)](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md)
Handles unstructured in-the-wild images by disentangling static geometry from dynamic appearance. Uses image-dependent embeddings. Choose for unconstrained, wild image collections.

### [TraGraph-GS](14454-tragraph-gs.md)
Uses graph-based adaptive spatial partitioning and regularization for large-scale scenes. Decouples local regions from global representation. Choose for vast, textureless, or large-scale scenes.

## Operating characteristics

**Time and Memory Complexity:**
Standard 3DGS training is memory-intensive due to per-Gaussian attributes (position, covariance, opacity, SH coefficients). The number of Gaussians grows during densification, increasing memory usage linearly with scene complexity [S2]. Half-precision optimization reduces register pressure and memory bandwidth requirements [S1].

**Measured Performance:**
On an NVIDIA A100 GPU, standard 3DGS achieves ~154 fps rendering with 19m 22s training time on Mip-NeRF 360 [S5]. Compact methods (e.g., using hash grids) can reach 254 fps with 17m 51s training time [S5]. On consumer GPUs (RTX4090), methods like EVolSplat achieve 83.81 fps with 10.41 GB memory usage on Waymo datasets [S2].

**Scaling:**
Large-scale scenes require spatial partitioning (e.g., TraGraph-GS) or voxel-based approaches to manage memory and rendering load [S9]. Multi-view training reduces gradient variance but increases computational overhead per step [S1].

**Strengths and Limitations:**
*   **Strengths:** Real-time rendering, high fidelity, differentiable optimization.
*   **Limitations:** High memory usage, sensitivity to camera distribution (scene extent bias), artifacts in sparse views or noisy environments [S6].

**When to Use:**
Use for high-quality novel view synthesis, SLAM, and applications requiring real-time rendering. Use variants for specific challenges (underwater, large-scale, semantic queries).

**When Not to Use:**
Avoid for scenes with extreme transient occlusions without distractor separation (DeSplat), or when storage is severely constrained without compression (Data Compression).

## Background

3D Gaussian Splatting was introduced as a real-time radiance field representation method [S1, S2]. It represents scenes as a set of 3D Gaussians optimized via differentiable rasterization. The method has been extended to handle various challenges, including underwater rendering [S8], large-scale scenes [S9], and semantic queries [S1]. The implementation details provided here are based on the `gsplat` library and various academic extensions [S1, S2, S5].

## Sources

[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S5] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S6] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S7] Augmented Radiance Field: A General Framework for Enhanced Gaussian Splatting, https://doi.org/10.48550/arXiv.2602.19916, fetched 2026-10-30 (unofficial)
[S8] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
[S9] TraGraph-GS: Trajectory Graph-based Gaussian Splatting for Arbitrary Large-Scale Scene Rendering., https://doi.org/10.1109/tpami.2026.3709368, fetched 2026-10-26
