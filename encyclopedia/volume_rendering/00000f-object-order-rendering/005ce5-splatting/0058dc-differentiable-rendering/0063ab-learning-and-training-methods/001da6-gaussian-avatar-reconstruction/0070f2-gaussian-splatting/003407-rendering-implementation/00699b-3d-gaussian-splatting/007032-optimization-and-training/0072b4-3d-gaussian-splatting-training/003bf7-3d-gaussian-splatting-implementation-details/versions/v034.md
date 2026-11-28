# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and variant algorithms used to train and render 3D Gaussian Splatting (3DGS) scenes. It addresses the practical challenges of differentiable rasterization, including memory management, precision stability, and scene initialization. While standard 3DGS assumes a static, well-lit environment with dense input views, these implementation details cover extensions for sparse views, underwater scattering, inverse rendering, and large-scale partitioning. The core distinction lies in the specific hyperparameters, loss formulations, and kernel-level optimizations (e.g., half-precision fetches) required to make the method viable in diverse and challenging scenarios.

## How it works

### Standard Training and Rasterization Configuration
The baseline implementation relies on the `gsplat` library [S2] using the 3DGUT formulation [S1]. The training process optimizes Gaussian parameters (position, covariance, opacity, color via Spherical Harmonics) using the Adam optimizer with a learning rate of $1 \times 10^{-3}$ [S2].

**Data Structures and Layouts:**
*   **Gaussian Attributes:** Each Gaussian $i$ stores position $\mathbf{p}_i$, scaling vector $\mathbf{s}_i$, rotation quaternion $\mathbf{q}_i$, opacity $\alpha_i$, and Spherical Harmonics coefficients $\mathbf{f}_c$ [S9].
*   **Volume Setup:** For large-scale urban scenes, an axis-aligned bounding box partitions the foreground. Typical dimensions include Height 12.8m, Width 32m, Length 64m, with a voxel size of 0.1m, resulting in a volume dimension of $128 \times 320 \times 640$ [S2]. For Waymo zero-shot inference, the range is adjusted to $(50\text{m}, 20\text{m}, 128\text{m})$ [S2].

**Optimization Steps:**
1.  **Initialization:** Initialize Gaussians from Structure-from-Motion (SfM) points.
2.  **Iteration:** Randomly select a single image from a random scene as supervision for each iteration [S2].
3.  **Forward Pass:** Render the image using differentiable rasterization. To reduce register pressure, half-precision (FP16) memory fetches are used for feature vectors during both forward and backward kernels [S1].
4.  **Loss Computation:** Compute the loss $\mathcal{L}$ using L1 and D-SSIM terms [S7].
5.  **Backward Pass:** Compute gradients and update parameters. An automatic scaling factor is applied to loss values to improve half-precision training stability [S1].
6.  **Neural Refinement (Optional):** For large-scale scenes, a refinement step may use tiny-cuda-nn's JIT-compiled cooperative vector MLPs, trained and evaluated in FP16, to slightly improve color fidelity [S1].

### Corrected Scene Extent
The default 3DGS scene extent $e_{\text{scene}}$ is proportional to the distance of the furthest camera from the average camera position $\bar{C}$ [S3]:
$$ e_{\text{scene}} = 1.1 \cdot \max_{i=1}^{N_{\text{cam}}} \|\bar{C} - C_i\|_2 $$
$$ \bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i $$
This causes bias if cameras are clustered (e.g., orbiting a small object), leading to incorrect splitting/pruning [S3]. The corrected extent $e'_{\text{scene}}$ uses the SfM point cloud:
$$ e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2 $$
where $N_{\text{SfM}}$ is the number of SfM points and $p_i$ is the $i$-th SfM point [S3].

### Selective Defocus Blurring
To handle spatially varying blur, a compact MLP predicts a per-Gaussian scale offset $\delta s_j$ [S4].
1.  Predict $\delta s_j$ for each Gaussian $j$.
2.  Add offset to base scale to transform covariance.
3.  Project transformed covariance to 2D space.
4.  Use as pixel-wise blur kernel during rasterization [S4].
If $\delta s_j$ is constant, it degrades to uniform Gaussian blur [S4].

### Denoise-GS (Sparse/Noisy Views)
A two-round self-supervised process combining Noise2Void (N2V) and 3DGS [S5]:
1.  **Round 1:** Train a UNet using N2V for denoising. Process denoised images through InstantSplat to initialize and train 3DGS.
2.  **Filtering:** Group and filter rendered results by viewpoint to ensure representativeness.
3.  **Round 2:** Fine-tune the UNet using a composite loss (low-frequency and perceptual detail consistency). Render again on selected viewpoints [S5].

### View-Dependent Opacity Kernel
Adds 5 learnable parameters to model specular reflectance lobes [S6, unofficial]. The opacity $\hat{\alpha}$ is:
$$ \hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos(\max(0, \min(\frac{2\theta}{T}, \pi)))+1}{2}\exp(\beta) $$
where $\alpha$ is base opacity, $\theta$ is the angle between view direction and lobe orientation, $T$ controls angular span, and $\beta$ modulates sharpness [S6, unofficial]. $T$ is initialized as $c\bar{\theta}_i/\pi$ based on viewpoint density, and $\beta$ to zero [S6, unofficial].

### Underwater Rendering (Gaussian Splashing)
Extends 3DGS with learnable backscatter parameters $B_d$ (direct attenuation), $B_\infty$ (infinite backscatter), and $B_b$ (backscatter coefficient) [S7].
**Rendering Formula:**
$$ C^{uw} = \sum_{i=1}^M c_i \alpha_i T_i e^{-B_d z} + B_\infty (1 - e^{-B_b z}) $$
where $T_i = \prod_{j=0}^{i-1} (1 - \alpha_j)$ [S7].
**Backscatter Estimation:**
1.  Divide depth map into 10 clusters.
2.  Identify pixels with RGB triplets in the bottom 1st percentile of ground-truth images within each cluster.
3.  Fit $B_\infty$ and $B_b$ using nonlinear least squares on pairs $(B_s, z)$ [S7].
4.  Re-estimate every 500 steps [S7].
**Depth Extraction:**
$$ z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i} $$
where $d_i$ is the depth of Gaussian $i$ [S7].
**Pitfall:** Camera sensor must have a linear response; gamma-corrected images cannot be used unless reversed [S7]. Bounds: $B_\infty \in [0, 1]$, $B_b \in [0, 5]$ [S7].

### Multi-Scale Constraint (Large-Scale Scenes)
Applies position-aware adaptive control based on viewer distance [S8].
$$ \gamma(\mu) = \begin{cases} 1 & \text{if } \|\mu - c\|^2 < 2r \\ \|\mu - c\|^2 / r - 1 & \text{if } \|\mu - c\|^2 \ge 2r \end{cases} $$
where $r$ is scene radius and $c$ is camera center [S8].
**Operations:**
*   Clone if $\max(s) < g \cdot \gamma(\mu)$.
*   Split if $\max(s) \ge g \cdot \gamma(\mu)$.
*   Eliminate if $\max(s) > b \cdot \gamma(\mu)$ [S8].
$g$ and $b$ are scale threshold parameters [S8].

### GS-IR (Inverse Rendering)
Extends Gaussian representation with normal vector $\mathbf{n}$ and material properties $\mathbf{M} = \{\text{albedo } a, \text{roughness } \rho, \text{metallicity } m\}$ [S9].
**Representation:**
*   Geometry $\mathbf{T} = \{\mathbf{p}, \mathbf{s}, \mathbf{q}, \mathbf{n}\}$ [S9].
*   Appearance $\mathbf{A} = \{\alpha, \mathbf{f}_c\}$ [S9].
**Training:**
1.  **Initial Stage:** Minimize color reconstruction loss $\mathcal{L}_c$ and normal loss $\mathcal{L}_n$ for 30K iterations to optimize $\mathbf{T}, \mathbf{A}$ [S9].
2.  **Decomposition Stage:** Fix $\mathbf{T}, \mathbf{A}$ and minimize decomposition loss $\mathcal{L}_d$ for 10K iterations to optimize $\mathbf{M}$ [S9].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. Ensures unbiased, ellipsoidal transmittance isosurfaces and prevents surface erosion [9624-continuous-attenuation-model-for-3d-gaussian-splatting.md].

### DAVIGS
Plug-and-play appearance modeling that mitigates floaters and color distortions by transforming rendered images using 3D consistent features queried from multi-resolution hash grids. Maintains real-time speed by discarding the appearance module after optimization [12021-davigs.md].

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses computational bottlenecks in 3DGS rendering by mapping software strategies to hardware constraints (e.g., Roofline model) or replacing standard sorting with hardware-friendly, comparison-free logic and sorting-free approximations [12180-sorting-and-rendering-pipeline-optimization.md].

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes 3DGS scenes to incorporate semantic language features (e.g., CLIP) alongside photometric rendering. Disables densification and pruning during language-aware training [12438-langsplat-training-procedure.md].

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps that offset SH colors. Uses dynamic texel allocation and splitting-based densification to avoid convergence issues [12606-content-aware-texturing-for-gaussian-splatting.md].

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Manages the lifecycle of 3D Gaussian primitives (initialization, densification, pruning, truncation) to balance fidelity and efficiency, focusing on optimization dynamics and structural constraints [12855-gaussian-representation-and-truncation.md].

### [DeSplat](12925-desplat.md)
Decomposes 3DGS into static scene Gaussians ($G_s$) and per-view distractor Gaussians ($G_d$) to remove transient occluders without external semantic models, relying solely on volume rendering and photometric loss [12925-desplat.md].

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Generates 3D representations from 360-degree equirectangular images by modifying the rasterizer for spherical rendering and using OpenSfM, which natively supports equirectangular inputs [12929-360-gaussian-splatting.md].

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint and storage overhead by compressing numerical representations, clustering parameters into codebooks, or replacing explicit attributes with neural fields [12943-data-compression-and-storage-optimization.md].

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of a discontinuous indicator function with respect to Bézier curve control points, enabling backpropagation in discontinuity-aware rendering [13137-b-zier-boundary-gradient-approximation.md].

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Enables 3DGS to represent sharp boundaries by introducing "pre-scissoring" along cubic Bézier curves and using Bézier-boundary gradient approximation [13138-discontinuity-aware-gaussian-splatting.md].

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Uses covariance-based feature augmentation to capture co-variation patterns, allowing smaller feature vectors and reduced model size compared to standard anchor methods [13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md].

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Adapts 3DGS to underwater environments by learning direct attenuation, infinite backscatter, and backscatter coefficient alongside geometry, using alternating optimization [13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md].

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Densifies 2D Gaussian kernels in image space using a differentiable rasterizer, then back-projects to 3D to initialize new 3D Gaussians, handling complex specular radiance [13985-error-driven-2d-gaussian-training.md].

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
Converts the volumetric 3DGS representation into explicit triangular meshes or point clouds for compatibility with standard 3D graphics pipelines [14095-mesh-extraction-and-representation.md].

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Reduces computational and memory overhead of view-dependent appearance by compressing SH coefficients, reducing polynomial degree, or optimizing SH gradient handling [14206-spherical-harmonics-optimization.md].

### [TraGraph-GS](14454-tragraph-gs.md)
Uses graph-based adaptive spatial partitioning and regularization constraints to handle large-scale scenes and textureless regions, decoupling local regions from a coarse global representation [14454-tragraph-gs.md].

### [Explicit 4D Gaussian Splatting](14584-explicit-4d-gaussian-splatting.md)
Achieves memory efficiency in dynamic scenes by separating Gaussians into static and dynamic categories and explicitly sampling positions/rotations at sparse timestamps [14584-explicit-4d-gaussian-splatting.md].

### [Color and Appearance Modeling](14740-color-and-appearance-modeling.md)
Extends standard 3DGS radiance representation to handle complex lighting and view-dependent effects by decoupling geometry from lighting or image-specific conditions [14740-color-and-appearance-modeling.md].

### [GS-IR](14852-gs-ir.md)
Leverages forward mapping volume rendering with 3D Gaussians to estimate scene geometry, surface material, and environment illumination for photorealistic relighting [14852-gs-ir.md].

## Operating characteristics

**Time and Memory Complexity:**
Standard 3DGS training is memory-intensive due to per-Gaussian attributes, often exceeding 1 GB for large scenes [12943-data-compression-and-storage-optimization.md]. Implementations like EVolSplat use 3D sparse convolution (torchsparse) to reduce memory consumption while maintaining fidelity [S2]. The use of half-precision (FP16) memory fetches and JIT-compiled MLPs reduces register pressure and improves training stability [S1].

**Measured Performance:**
EVolSplat achieves 83.81 FPS on Waymo data with 10.41 GB memory usage, outperforming MVSNeRF (12.03 FPS, 16.14 GB) and PixelSplat (70.46 FPS, 26.75 GB) [S2]. GS-IR runs on a single V100 GPU, with 30K iterations for initial stage and 10K for decomposition [S9].

**Strengths and Limitations:**
*   **Strengths:** Real-time rendering speeds [S1, S9], robustness to sparse views with Denoise-GS [S5], ability to handle underwater scattering [S7], and inverse rendering capabilities [S9].
*   **Limitations:** Standard 3DGS struggles with large-scale scenes without multi-scale constraints [S8] or graph-based partitioning [S8]. Default scene extent calculation is biased by camera distribution [S3]. Gamma-corrected images cannot be used for underwater backscatter estimation [S7].

**When to Use:**
Use standard 3DGS for dense-view, well-lit scenes. Use variants for specific challenges: Denoise-GS for sparse/noisy views, GS-IR for inverse rendering, Underwater GS for aquatic environments, and TraGraph-GS for large-scale scenes.

## Background

3D Gaussian Splatting was introduced as a real-time explicit scene representation method [26, cited in S7]. The implementation details described here stem from various extensions and optimizations developed to address the limitations of the original method in specific domains such as urban view synthesis [S2], underwater environments [S7], inverse rendering [S9], and large-scale scenes [S8]. The `gsplat` library [S2] and 3DGUT formulation [S1] provide the foundational CUDA kernels for efficient differentiable rasterization.

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
