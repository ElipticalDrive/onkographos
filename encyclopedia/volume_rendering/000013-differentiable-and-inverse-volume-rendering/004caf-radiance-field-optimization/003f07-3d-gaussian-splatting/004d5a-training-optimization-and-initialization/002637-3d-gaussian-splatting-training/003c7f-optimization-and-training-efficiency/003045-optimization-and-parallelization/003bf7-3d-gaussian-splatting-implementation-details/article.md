# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and variant algorithms required to train and render 3D Gaussian Splatting (3DGS) scenes effectively. It addresses the gap between the theoretical formulation of differentiable volume rendering and the practical constraints of GPU memory, precision, and scene geometry. Key distinctions from standard implementations include the use of half-precision rasterization for register efficiency, corrected scene extent calculations to prevent bias from camera distribution, and specialized handling for defocus blur, noise, and underwater scattering.

## How it works

### Standard Training Configuration
The baseline training process relies on the Adam optimizer with a learning rate of $1 \times 10^{-3}$ [S2]. The Gaussian Rasterization Library `gsplat` is typically used, often combined with `torchsparse` for 3D sparse convolution operations [S2]. Spherical Harmonics (SH) degree is commonly set to 1 to balance view-dependent appearance quality and computational cost [S2].

**Data Structures and Memory:**
To reduce register pressure during rasterization, feature vectors are fetched using half-precision (FP16) memory accesses for both forward and backward kernels [S1]. Neural network components, such as appearance MLPs, are implemented using JIT-compiled cooperative vector MLPs (e.g., via `tiny-cuda-nn`) and evaluated in FP16 [S1]. An automatic scaling factor is applied to loss values to maintain stability during half-precision training [S1].

**Training Loop:**
1.  **Input:** A set of images with known camera poses and an initial point cloud (e.g., from SfM).
2.  **Initialization:** Initialize 3D Gaussians from the point cloud.
3.  **Iteration:**
    *   Randomly select a single image from the training set [S2].
    *   Render the image using the differentiable rasterizer.
    *   Compute loss (e.g., L1 + SSIM) against the ground truth image.
    *   Backpropagate gradients to update Gaussian parameters (position, covariance, opacity, color/SH) and any auxiliary parameters (e.g., backscatter coefficients) [S2].
    *   Apply densification (cloning/splitting) and pruning based on gradient thresholds and opacity [S3].

### Corrected Scene Extent
Standard 3DGS calculates scene extent $e_{\text{scene}}$ based on camera positions, which biases optimization if cameras are clustered (e.g., orbiting a small object in a large environment) [S3]. The corrected method uses the Structure-from-Motion (SfM) point cloud [S3].

**Formulas:**
1.  Average camera position:
    $$ \bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i $$
    where $N_{\text{cam}}$ is the number of camera views and $C_i$ is the position of the $i$-th camera [S3].
2.  Corrected scene extent:
    $$ e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2 $$
    where $N_{\text{SfM}}$ is the number of SfM points and $p_i$ is the $i$-th SfM point [S3].

**Pitfall:** Using the default extent formula leads to incorrect splitting/pruning for large outdoor scenes captured with clustered cameras [S3].

### Selective Defocus Blurring
To handle spatially varying blur (e.g., defocus), a compact MLP predicts a per-Gaussian scale offset $\delta s_j$ [S4].

**Steps:**
1.  Predict $\delta s_j$ for each Gaussian $j$ using the MLP.
2.  Add $\delta s_j$ to the base scale to transform the Gaussian covariance.
3.  Project the transformed covariance to 2D space.
4.  Use the transformed covariance as a pixel-wise blur kernel during rasterization [S4].

**Pitfall:** If $\delta s_j$ is constant for all Gaussians, the method degrades to a uniform Gaussian blur, failing to model spatially varying blurriness [S4].

### Denoise-GS (Sparse-View Training)
This method combines unsupervised image denoising with 3DGS training in two rounds [S5].

**Steps:**
1.  **Round 1:** Train a UNet using Noise2Void (N2V) for denoising on input images.
2.  Process denoised images through InstantSplat to initialize and train 3DGS.
3.  Render results and group/filter by viewpoint to ensure representativeness.
4.  **Round 2:** Fine-tune the UNet using a composite loss (low-frequency consistency + perceptual detail consistency).
5.  Render again on selected viewpoints to generate denoised views [S5].

**Comparison:** Unlike supervised methods, N2V requires no paired clean data, making it robust to non-Gaussian noise in sparse views where N2N (Noise2Noise) fails due to lack of paired noisy samples [S5].

### View-Dependent Opacity Kernel
To model specular reflectance, a transparent Gaussian kernel uses a half-period cosine function for view-dependent opacity [S6, unofficial].

**Formula:**
$$ \hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos(\max(0, \min(\frac{2\theta}{T}, \pi)))+1}{2}\exp(\beta) $$
where $\alpha$ is base opacity, $\theta$ is the angle between view direction and lobe orientation, $T$ controls angular span, and $\beta$ modulates sharpness [S6, unofficial].

**Parameters:**
*   $T$: Initialized as $c\bar{\theta}_i/\pi$ based on viewpoint density.
*   $\beta$: Initialized to zero.
*   $\theta$: Angle between viewing direction and lobe central orientation.

**Pitfall:** As $\theta$ increases, transparency diminishes to zero, preventing artifacts outside the lobe's influence [S6, unofficial].

### Underwater Rendering (Gaussian Splashing)
Adapts 3DGS for underwater scenes by learning direct attenuation ($B_d$), infinite backscatter ($B_\infty$), and backscatter coefficient ($B_b$) [S7].

**Rendering Formula:**
$$ C_{uw} = \sum_{i=1}^M c_i \alpha_i T_i e^{-B_d z} + B_\infty (1 - e^{-B_b z}) $$
where $T_i = \prod_{j=0}^{i-1} (1 - \alpha_j)$ and $z$ is the depth of Gaussian $i$ [S7].

**Backscatter Estimation:**
1.  Divide depth map into 10 clusters.
2.  Identify pixels with RGB triplets in the bottom 1st percentile of ground truth images within each cluster.
3.  Fit $B_\infty$ and $B_b$ using nonlinear least squares on pairs $(B_s, z)$, where $B_s$ is the estimated backscatter [S7].
4.  Re-estimate parameters every 500 steps [S7].

**Depth Extraction:**
$$ z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i} $$
where $d_i$ is the depth of Gaussian $i$ [S7].

**Pitfall:** Camera sensor must have a linear response; in-camera processed .jpg or gamma-corrected images cannot be used unless non-linearities are reversed [S7].

### Multi-Scale Constraint
For large-scale scenes, a position-aware point adaptive control adjusts Gaussian scales based on viewer distance [S8].

**Formula:**
$$ \gamma(\mu) = \begin{cases} 1 & \text{if } \|\mu - c\|^2 < 2r \\ \|\mu - c\|^2 / r - 1 & \text{if } \|\mu - c\|^2 \ge 2r \end{cases} $$
where $r$ is scene radius and $c$ is camera center [S8].

**Operations:**
*   Clone if $\max(s) < g \cdot \gamma(\mu)$.
*   Split if $\max(s) \ge g \cdot \gamma(\mu)$.
*   Eliminate if $\max(s) > b \cdot \gamma(\mu)$.
where $g$ and $b$ are scale threshold parameters [S8].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. Ensures unbiased, ellipsoidal transmittance isosurfaces, preventing surface erosion and multi-view inconsistency seen in prior approximations like GGGS [9624-continuous-attenuation-model-for-3d-gaussian-splatting.md].

### DAVIGS
A plug-and-play appearance modeling method that mitigates floaters and color distortions by transforming rendered images using 3D consistent features queried from multi-resolution hash grids. Decouples global view-dependent variations from local spatial changes, maintaining real-time speeds by discarding the appearance module after optimization [12021-davigs.md].

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses memory bandwidth and compute throughput bottlenecks by mapping software strategies to hardware constraints (Roofline model) or replacing standard sorting with hardware-friendly, comparison-free logic and sorting-free approximations [12180-sorting-and-rendering-pipeline-optimization.md].

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes 3DGS scenes to incorporate semantic language features (e.g., CLIP) alongside photometric rendering. Disables densification and pruning during language-aware training to support semantic queries [12438-langsplat-training-procedure.md].

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps that offset SH colors. Uses dynamic texel allocation and splitting-based densification to avoid convergence issues associated with cloning textured primitives [12606-content-aware-texturing-for-gaussian-splatting.md].

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
A neural field mapping rays to distance and visibility for constant-time, constant-memory ray queries (shadows, AO, GI). Eliminates the need for mesh extraction or BVH traversal, scaling independently of scene size [12854-directed-distance-field-ddf-for-gaussian-splatting.md].

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Modifies spatial extent, density, and survival criteria using algebraic decay functions, opacity-independent culling, and geometry-guided proliferation to improve geometric fidelity and reduce redundancy [12855-gaussian-representation-and-truncation.md].

### [DeSplat](12925-desplat.md)
Decomposes 3DGS into static scene Gaussians and per-view distractor Gaussians to remove transient occluders (pedestrians, vehicles) without external semantic models. Relies solely on volume rendering and photometric loss [12925-desplat.md].

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Generates 3D representations from 360-degree equirectangular images by modifying the rasterizer for spherical rendering and using OpenSfM (which supports equirectangular inputs) instead of Colmap [12929-360-gaussian-splatting.md].

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint by compressing numerical representations, clustering parameters into codebooks, or replacing explicit attributes with neural fields. Focuses on data density rather than rendering throughput [12943-data-compression-and-storage-optimization.md].

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Places isometric 3D Gaussians at fixed voxel grid centroids for efficient CT volume extraction and differentiable X-ray rendering. Fixes positions and removes color coefficients, retaining only opacity, scale, and rotation [13071-voxel-based-gaussian-splatting.md].

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of discontinuous indicator functions with respect to Bézier curve control points, enabling backpropagation in discontinuity-aware rendering [13137-b-zier-boundary-gradient-approximation.md].

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Enables 3DGS to represent sharp boundaries by introducing "pre-scissoring" along cubic Bézier curves and using Bézier-boundary gradient approximation to maintain differentiability [13138-discontinuity-aware-gaussian-splatting.md].

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Uses covariance-based feature augmentation to capture co-variation patterns, achieving superior quality with smaller feature vectors compared to standard anchor methods like Scaffold-GS [13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md].

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Establishes a continuous scene representation via kernel regression, replacing discrete opacity values with Gaussian kernel functions. Enables direct mesh extraction and reduces storage/training memory [13979-gaussian-voxel-kernel-functions-gvkf.md].

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Integrates ocean optics models (direct attenuation, backscatter) into 3DGS. Uses alternating optimization to refine depth and medium parameters simultaneously, improving quality at large distances [13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md].

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Densifies 2D Gaussian kernels in image space using a differentiable rasterizer, then back-projects to 3D. Solves initialization issues for complex specular radiance where standard 3D initialization is insufficient [13985-error-driven-2d-gaussian-training.md].

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
[14095-mesh-extraction-and-representation.md]

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Reduces memory bandwidth and computational costs of high-degree SH coefficients by modifying degree, distilling knowledge, or quantizing coefficients into shared palettes [14206-spherical-harmonics-optimization.md].

### [TraGraph-GS](14454-tragraph-gs.md)
Uses graph-based adaptive spatial partitioning and regularization for large-scale scenes. Decouples scenes into local regions and a coarse global representation, using a camera connectivity graph to ensure balanced subgraphs [14454-tragraph-gs.md].

### [Explicit 4D Gaussian Splatting](14584-explicit-4d-gaussian-splatting.md)
Achieves memory efficiency in dynamic scenes by separating Gaussians into static and dynamic categories, explicitly sampling positions/rotations at sparse timestamps, and interpolating for continuous motion [14584-explicit-4d-gaussian-splatting.md].

### [3D Gaussian Particle Approximation of VDB Datasets](14723-3d-gaussian-particle-approximation-of-vdb-datasets.md)
Converts OpenVDB volumetric datasets into non-overlapping 3D Gaussian particles for memory-efficient, lossy representation with LOD support. Enforces deterministic spatial partitioning derived from OpenVDB leaf nodes [14723-3d-gaussian-particle-approximation-of-vdb-datasets.md].

### [Color and Appearance Modeling](14740-color-and-appearance-modeling.md)
Extends standard 3DGS radiance representation to handle complex lighting and view-dependent effects by decoupling geometry from lighting or using dynamic appearance embeddings [14740-color-and-appearance-modeling.md].

## Operating characteristics

**Time and Memory Complexity:**
Standard 3DGS training is memory-intensive due to per-Gaussian attributes (position, covariance, opacity, SH coefficients). Half-precision rasterization reduces register pressure and memory bandwidth usage [S1]. Large-scale scenes require spatial partitioning (e.g., TraGraph-GS) or compression to manage memory [S8, 12943-data-compression-and-storage-optimization.md].

**Measured Performance:**
In urban view synthesis (KITTI-360, Waymo), EVolSplat achieves high fidelity with reduced memory consumption compared to MVSNeRF and MuRF, leveraging 3D sparse convolution [S2].

**Strengths and Limitations:**
*   **Strengths:** Real-time rendering, high fidelity, differentiable optimization.
*   **Limitations:** Sensitive to camera distribution (addressed by Corrected Scene Extent [S3]), struggles with defocus blur (addressed by Selective Defocus [S4]), noise in sparse views (addressed by Denoise-GS [S5]), and underwater scattering (addressed by Gaussian Splashing [S7]).

**When to Use:**
Use for high-quality novel view synthesis, especially with standard perspective cameras. Use variants for specific conditions: underwater (Gaussian Splashing), sparse/noisy views (Denoise-GS), defocus (Selective Defocus), or large-scale scenes (TraGraph-GS, Multi-Scale Constraint).

## Background

3D Gaussian Splatting was introduced as a real-time differentiable renderer using 3D Gaussian primitives [S2, S7]. Subsequent research has focused on implementation details to improve stability, accuracy, and applicability to diverse environments. Key developments include half-precision rasterization for efficiency [S1], corrected scene extent for unbiased optimization [S3], and specialized models for defocus [S4], noise [S5], specular reflectance [S6, unofficial], underwater scattering [S7], and large-scale scenes [S8].

## Sources

[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S5] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S6] Augmented Radiance Field: A General Framework for Enhanced Gaussian Splatting, https://doi.org/10.48550/arXiv.2602.19916, fetched 2026-10-30 (unofficial)
[S7] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
[S8] TraGraph-GS: Trajectory Graph-based Gaussian Splatting for Arbitrary Large-Scale Scene Rendering., https://doi.org/10.1109/tpami.2026.3709368, fetched 2026-10-26
