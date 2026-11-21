# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and variant methodologies required to train and render 3D Gaussian Splatting (3DGS) effectively. It addresses the gap between the theoretical formulation of differentiable volume rendering and the practical constraints of GPU memory, precision, and scene geometry. It sets itself apart from general optimization articles by providing concrete hyperparameters, data structure layouts, and specific algorithmic modifications for handling edge cases such as sparse views, underwater scattering, and inverse rendering.

## How it works

### Standard Training Configuration
The baseline implementation relies on the `gsplat` library [S2] using the 3DGUT formulation [S1].

**Inputs:**
*   Sparse point cloud from Structure-from-Motion (SfM).
*   Camera poses and intrinsics.
*   Training images.

**Outputs:**
*   Optimized set of 3D Gaussians with parameters $\Theta = \{\mathbf{p}, \mathbf{s}, \mathbf{q}, \alpha, \mathbf{c}_{SH}\}$.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian stores position $\mathbf{p} \in \mathbb{R}^3$, scaling $\mathbf{s} \in \mathbb{R}^3$, rotation $\mathbf{q} \in \mathbb{R}^4$ (unit quaternion), opacity $\alpha \in \mathbb{R}$, and Spherical Harmonics (SH) coefficients $\mathbf{c}_{SH}$.
*   **Volume Grid:** For large-scale scenes, a 3D sparse convolution grid (e.g., `torchsparse`) may be used to partition the foreground [S2]. Typical dimensions for urban scenes (KITTI-360) are Height 12.8m, Width 32m, Length 64m, with voxel size 0.1m, resulting in a volume dimension of $128 \times 320 \times 640$ [S2].

**Optimization Steps:**
1.  **Initialization:** Initialize Gaussians from the SfM point cloud.
2.  **Iteration:**
    *   Randomly select a single image from the training set [S2].
    *   Render the image using the differentiable rasterizer.
    *   Compute loss $\mathcal{L}$ against the ground truth image.
    *   Backpropagate gradients to update Gaussian parameters.
3.  **Densification/Pruning:** Periodically split, clone, or prune Gaussians based on gradient thresholds and opacity [S3].

**Parameters:**
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** $1 \times 10^{-3}$ [S2].
*   **SH Degree:** 1 (15 coefficients per channel) [S2].
*   **Loss Weights:** $\lambda_r = 0.2, \lambda_e = 0.1$ (for specific variants like EVolSplat) [S2].

**Pitfalls:**
*   **Precision:** Half-precision (FP16) training requires an automatic scaling factor for loss values to maintain stability [S1].
*   **Scene Extent:** Default scene extent calculation based on camera positions can bias densification if cameras are clustered [S3].

### Custom CUDA Kernel Implementation
To achieve real-time performance, custom CUDA kernels are implemented within the `gsplat` framework [S1].

**Implementation Details:**
1.  **Forward/Backward Logic:** Custom kernels handle the projection of 3D Gaussians to 2D tiles and the alpha-blending accumulation.
2.  **Memory Optimization:** Half-precision memory fetches are used for feature vectors during rasterization (both forward and backward passes) to reduce register pressure [S1].
3.  **Neural Components:** If MLPs are used (e.g., for appearance modeling), they are implemented using `tiny-cuda-nn`'s JIT-compiled cooperative vector MLPs, trained and evaluated in FP16 [S1].

### Variant: Corrected Scene Extent
Standard 3DGS calculates scene extent $e_{\text{scene}}$ based on camera positions, which biases densification if cameras are orbiting a small object in a large environment [S3].

**Formula:**
The corrected scene extent $e'_{\text{scene}}$ is calculated using the SfM point cloud:
$$ \bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i $$
$$ e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2 $$
Where $N_{\text{cam}}$ is the number of cameras, $C_i$ is the $i$-th camera position, $N_{\text{SfM}}$ is the number of SfM points, and $p_i$ is the $i$-th SfM point [S3].

### Variant: Selective Defocus Blurring
To handle spatially varying blur, a per-Gaussian scale offset $\delta s_j$ is predicted [S4].

**Steps:**
1.  Predict $\delta s_j$ for each Gaussian $j$ using a compact MLP.
2.  Add $\delta s_j$ to the base scale to transform the covariance matrix.
3.  Project the transformed covariance to 2D space.
4.  Use the projected shape as a pixel-wise blur kernel during rasterization [S4].

**Pitfall:** If $\delta s_j$ is constant for all Gaussians, the method degrades to a uniform Gaussian blur [S4].

### Variant: Denoise-GS (Sparse/Noisy Views)
A two-round self-supervised denoising pipeline [S5].

**Steps:**
1.  **Round 1:** Train a UNet using Noise2Void (N2V) for unsupervised denoising.
2.  **Round 1:** Process denoised images through InstantSplat to initialize and train 3DGS.
3.  **Round 1:** Group and filter rendered results by viewpoint to ensure representativeness.
4.  **Round 2:** Fine-tune the UNet using a composite loss (low-frequency consistency + perceptual detail consistency).
5.  **Round 2:** Render again on selected viewpoints to generate denoised views [S5].

### Variant: View-Dependent Opacity Kernel
Models specular reflectance lobes using a view-dependent opacity kernel [S6, unofficial].

**Formula:**
$$ \hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos(\max(0, \min(\frac{2\theta}{T}, \pi)))+1}{2}\exp(\beta) $$
Where $\alpha$ is base opacity, $\theta$ is the angle between view direction and lobe orientation, $T$ controls angular span, and $\beta$ modulates sharpness [S6, unofficial].

**Parameters:**
*   $T$: Initialized as $c\bar{\theta}_i/\pi$ based on viewpoint density.
*   $\beta$: Initialized to zero.
*   **Cost:** Adds 5 learnable parameters per Gaussian [S6, unofficial].

**Pitfall:** As $\theta$ increases, transparency diminishes to zero, preventing artifacts outside the lobe [S6, unofficial].

### Variant: Underwater Rendering (Gaussian Splashing)
Extends 3DGS to model underwater scattering with learnable parameters $B_d$ (direct attenuation), $B_\infty$ (infinite backscatter), and $B_b$ (backscatter coefficient) [S7].

**Rendering Formula:**
$$ C_{uw} = \sum_{i=1}^M c_i \alpha_i T_i e^{-B_d z} + B_\infty (1 - e^{-B_b z}) $$
Where $T_i = \prod_{j=0}^{i-1} (1 - \alpha_j)$ is the transmittance, and $z$ is the depth [S7].

**Backscatter Estimation Steps:**
1.  Divide the depth map into 10 evenly-spaced clusters.
2.  Identify pixels in the ground-truth image with RGB triplets below the bottom 1st percentile within each cluster.
3.  Use these triplets as an overestimate of backscatter $B_s$.
4.  Fit $B_\infty$ and $B_b$ using nonlinear least squares on pairs $(B_s, z)$.
5.  Re-estimate parameters every 500 optimization steps [S7].

**Depth Extraction:**
$$ z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i} $$
Where $d_i$ is the depth of Gaussian $i$ [S7].

**Pitfall:** Camera sensor must have a linear response. In-camera processed .jpg or gamma-corrected images cannot be used unless non-linearities are reversed [S7].

**Parameter Bounds:**
*   $B_\infty \in [0, 1]$
*   $B_b \in [0, 5]$ [S7]

### Variant: Multi-Scale Constraint (TraGraph-GS)
Applies position-aware adaptive control to handle large-scale scenes [S8].

**Formula:**
$$ \gamma(\mu) = \begin{cases} 1 & \text{if } \|\mu - c\|^2 < 2r \\ \|\mu - c\|^2 / r - 1 & \text{if } \|\mu - c\|^2 \ge 2r \end{cases} $$
Where $r$ is scene radius, $c$ is camera center, and $\mu$ is Gaussian position [S8].

**Steps:**
1.  If $\max(s) < g \cdot \gamma(\mu)$, perform cloning.
2.  If $\max(s) \ge g \cdot \gamma(\mu)$, execute splitting.
3.  Eliminate Gaussians where $\max(s) > b \cdot \gamma(\mu)$.
Where $g$ and $b$ are scale threshold parameters [S8].

### Variant: GS-IR (Inverse Rendering)
Extends 3DGS to estimate geometry, material, and illumination [S9].

**Representation:**
*   **Geometry:** $\mathbf{T} = \{\mathbf{p}, \mathbf{s}, \mathbf{q}, \mathbf{n}\}$, where $\mathbf{n}$ is the normal vector.
*   **Material:** $\mathbf{M} = \{a, \rho, m\}$, where $a$ is albedo, $\rho$ is roughness, $m$ is metallicity.
*   **Appearance:** $\mathbf{A} = \{\alpha, \mathbf{f}_c\}$ (vanilla 3DGS parameters) [S9].

**Training Steps:**
1.  **Initial Stage (30K iters):** Minimize color reconstruction loss $\mathcal{L}_c$ and normal loss $\mathcal{L}_n$ to optimize $\mathbf{T}$ and $\mathbf{A}$.
2.  **Decomposition Stage (10K iters):** Fix $\mathbf{T}$ and $\mathbf{A}$; minimize decomposition loss $\mathcal{L}_d$ to optimize $\mathbf{M}$ [S9].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. Ensures unbiased, ellipsoidal transmittance isosurfaces and prevents surface erosion.

### [DAVIGS](12021-davigs.md)
Plug-and-play appearance modeling that decouples global view-dependent variations (e.g., daylight) from local spatial changes using multi-resolution hash grids. Maintains real-time speed by discarding the appearance module after optimization.

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses memory bandwidth and compute throughput bottlenecks by mapping software strategies to hardware constraints (Roofline model) and using comparison-free sorting logic.

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes 3DGS to incorporate semantic language features (e.g., CLIP) alongside photometric rendering. Disables densification/pruning during language-aware training.

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps and dynamic texel allocation to adapt appearance without pretrained models.

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Focuses on spatial decay profiles (e.g., Inverse Multiquadric), truncation thresholds, and structural constraints (atomization, voxel-fixing) to manage Gaussian lifecycle and culling.

### [DeSplat](12925-desplat.md)
Decomposes Gaussians into static ($G_s$) and per-view distractor ($G_d$) sets to remove transient occluders (pedestrians) without external semantic models.

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Adapts 3DGS for 360-degree equirectangular images by modifying the rasterizer for spherical rendering and using OpenSfM instead of Colmap.

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint via numerical compression, codebook clustering, or neural field replacement, focusing on data density rather than rendering throughput.

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of discontinuous indicator functions with respect to Bézier control points, enabling backpropagation for discontinuity-aware rendering.

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Introduces "pre-scissoring" along cubic Bézier curves to render sharp boundaries, combined with Bézier-boundary gradient approximation.

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Uses covariance-based feature augmentation to capture co-variation patterns, allowing smaller feature vectors and reduced model size compared to standard anchor methods.

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Integrates ocean optics models (direct attenuation, backscatter) and uses alternating optimization to refine depth and medium parameters simultaneously.

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Densifies 2D Gaussians in image space using a differentiable rasterizer, then back-projects to 3D to initialize new 3D Gaussians for complex specular radiance.

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
Converts volumetric Gaussians into explicit triangular meshes or point clouds for compatibility with standard 3D graphics pipelines.

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Improves efficiency and accuracy of directional appearance representation, targeting the computational cost and gradient dynamics of high-degree SH coefficients.

### [TraGraph-GS](14454-tragraph-gs.md)
Uses graph-based adaptive spatial partitioning and camera connectivity graphs to handle large-scale scenes and textureless regions, eliminating boundary artifacts.

### [Explicit 4D Gaussian Splatting](14584-explicit-4d-gaussian-splatting.md)
Separates Gaussians into static and dynamic categories, explicitly sampling positions/rotations at sparse timestamps for memory-efficient dynamic scene rendering.

### [Color and Appearance Modeling](14740-color-and-appearance-modeling.md)
Extends radiance representation to handle complex lighting and view-dependent effects by decoupling geometry from lighting or image-specific conditions.

### [GS-IR](14852-gs-ir.md)
Inverse rendering approach estimating geometry, surface material, and environment illumination using forward mapping volume rendering for real-time efficiency.

## Operating characteristics

**Time and Memory Complexity:**
*   Standard 3DGS is $O(N)$ with respect to the number of Gaussians per pixel, but sorting and attribute fetching can be memory-bandwidth bound [S1].
*   Custom CUDA kernels with half-precision fetches reduce register pressure and improve throughput [S1].
*   Large-scale scenes (e.g., urban) require significant memory; sparse convolution grids can mitigate this but add overhead [S2].

**Measured Performance:**
*   On NVIDIA V100, methods like EVolSplat achieve ~80 FPS with 10-16 GB memory usage for urban scenes [S2].
*   Consumer GPUs (e.g., RTX 4090) can train EVolSplat but may struggle with full-resolution transformer-based baselines [S2].

**Strengths:**
*   Real-time rendering speeds [S1].
*   High fidelity with relatively simple representation [S2].
*   Flexibility to incorporate various physical models (underwater, inverse rendering) [S7, S9].

**Limitations:**
*   Sensitive to initialization and scene extent calculation [S3].
*   Standard SH representation can be memory-intensive for high-degree coefficients [S2].
*   Requires linear camera response for accurate physical modeling (e.g., underwater) [S7].

## Background

3D Gaussian Splatting was introduced by Kerbl et al. [S1, S2] as a real-time explicit scene representation. It builds on the concept of differentiable rasterization of 3D Gaussians. The method has since been extended to handle various challenges including sparse views [S5], underwater environments [S7], inverse rendering [S9], and large-scale scenes [S8]. The implementation details described here reflect the evolution of the method to address hardware constraints and physical accuracy.

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
