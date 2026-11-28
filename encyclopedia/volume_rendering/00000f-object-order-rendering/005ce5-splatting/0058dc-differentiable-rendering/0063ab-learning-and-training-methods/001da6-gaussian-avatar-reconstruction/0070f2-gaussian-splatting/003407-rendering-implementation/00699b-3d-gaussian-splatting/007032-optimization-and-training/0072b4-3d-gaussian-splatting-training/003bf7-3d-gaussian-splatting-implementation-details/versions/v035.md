# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation strategies, parameter configurations, and variant algorithms required to train and render 3D Gaussian Splatting (3DGS) scenes effectively. It addresses the gap between the theoretical differentiable volume rendering formulation and practical software engineering, focusing on memory efficiency, optimization stability, and handling specific scene conditions (e.g., underwater, sparse views, large-scale). It sets itself apart from general optimization articles by providing concrete hyperparameters, data structures, and custom kernel logic (e.g., half-precision fetches, specific loss scaling) necessary for a working implementation.

## How it works

### Standard Implementation Parameters
The baseline implementation relies on the `gsplat` library [S2] using the 3DGUT formulation [S1].

**Data Structures and Layouts:**
*   **Gaussian Attributes:** Each Gaussian $i$ stores position $\mathbf{p}_i$, scaling vector $\mathbf{s}_i$, rotation quaternion $\mathbf{q}_i$, opacity $\alpha_i$, and Spherical Harmonics (SH) coefficients $\mathbf{f}_c$.
*   **Memory Fetch:** To reduce register pressure during rasterization, feature vectors are fetched using half-precision (FP16) memory accesses for both forward and backward kernels [S1].
*   **Neural Networks:** If auxiliary networks (e.g., for appearance or depth) are used, they are implemented using tiny-cuda-nn's JIT-compiled cooperative vector MLPs, trained and evaluated in FP16 [S1].

**Optimization Loop:**
1.  **Input:** A set of training images with known camera poses.
2.  **Iteration:** Randomly select a single image from the training set [S2].
3.  **Forward Pass:** Render the image using the differentiable rasterizer.
4.  **Loss Calculation:** Compute the loss $\mathcal{L}$ between the rendered image and the ground truth.
    *   **Loss Scaling:** Apply an automatic scaling factor to the loss values to improve training stability in half-precision [S1].
    *   **Loss Function:** Typically a combination of L1 loss and D-SSIM:
        $$ \mathcal{L} = (1 - \lambda_1) \| I^{gt} - I^r(\Theta) \|_{\ell_1} + \lambda_1 L_{D-SSIM}(I^{gt}, I^r(\Theta)) $$
        where $\lambda_1 \in [0,1]$ is a user-defined weight [S7].
5.  **Backward Pass:** Compute gradients $\nabla_\Theta \mathcal{L}$ with respect to Gaussian parameters.
6.  **Update:** Update parameters using the Adam optimizer with a learning rate of $1 \times 10^{-3}$ [S2].
7.  **Densification/Pruning:** Apply density control strategies (see Subtopics) based on gradient thresholds and scene extent.

**Typical Parameters:**
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** $1 \times 10^{-3}$ [S2].
*   **SH Degree:** 1 (for simplicity and speed in some implementations) [S2].
*   **Volume Dimensions (Example):** For urban scenes, a voxel size of 0.1m might result in a volume of $128 \times 320 \times 640$ (Height 12.8m, Width 32m, Length 64m) [S2].

**Edge Cases and Pitfalls:**
*   **Half-Precision Stability:** FP16 training requires loss scaling to prevent gradient underflow [S1].
*   **Scene Extent Bias:** Default scene extent calculation based on camera positions can bias densification if cameras are clustered (e.g., orbiting a small object) [S3].

### Variant: Corrected Scene-Extent
To mitigate the bias described above, the scene extent $e'_{\text{scene}}$ is calculated based on the Structure-from-Motion (SfM) point cloud rather than camera positions [S3].

**Formula:**
1.  Calculate average camera position $\bar{C}$:
    $$ \bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i $$
2.  Calculate corrected scene extent:
    $$ e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2 $$
    where $N_{\text{SfM}}$ is the number of SfM points and $p_i$ is the $i$-th SfM point [S3].

### Variant: Selective Defocus Blurring
This method handles spatially varying blur by predicting a per-Gaussian scale offset $\delta s_j$ [S4].

**Steps:**
1.  Predict $\delta s_j$ for each Gaussian $j$ using a compact MLP.
2.  Add $\delta s_j$ to the base scale to transform the covariance matrix.
3.  Project the transformed covariance to 2D space.
4.  Use the projected 2D covariance as a pixel-wise blur kernel during rasterization [S4].

**Pitfall:** If $\delta s_j$ is predicted as a constant for all Gaussians, the method degrades to a uniform Gaussian blur, failing to model spatially varying blurriness [S4].

### Variant: Denoise-GS
A self-supervised pipeline for sparse-view and noisy scenes [S5].

**Steps:**
1.  **Round 1:** Train a UNet using Noise2Void (N2V) for unsupervised denoising.
2.  **Round 1:** Process denoised images through InstantSplat to initialize and train 3DGS.
3.  **Round 1:** Group and filter rendered results by viewpoint to ensure representativeness.
4.  **Round 2:** Fine-tune the UNet using a composite loss (low-frequency consistency + perceptual detail consistency).
5.  **Round 2:** Render again on selected viewpoints to generate denoised views [S5].

### Variant: View-Dependent Opacity Kernel
Models specular reflectance lobes using a view-dependent opacity $\hat{\alpha}$ [S6, unofficial].

**Formula:**
$$ \hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos(\max(0, \min(\frac{2\theta}{T}, \pi)))+1}{2}\exp(\beta) $$
where:
*   $\alpha$: Base opacity.
*   $\theta$: Angle between view direction and lobe central orientation.
*   $T$: Angular span of the lobe (initialized as $c\bar{\theta}_i/\pi$).
*   $\beta$: Sharpness modulator (initialized to 0) [S6, unofficial].

**Pitfall:** As $\theta$ increases, transparency diminishes to zero, preventing artifacts outside the lobe's influence [S6, unofficial]. This adds 5 learnable parameters per Gaussian [S6, unofficial].

### Variant: Underwater Rendering (Gaussian Splashing)
Adapts 3DGS for underwater scenes by estimating backscatter parameters $B_\infty$ and $B_b$ [S7].

**Backscatter Estimation Algorithm:**
1.  Divide the current depth map into 10 evenly-spaced clusters from $\min(z)$ to $\max(z)$.
2.  In each cluster, identify pixels in the ground-truth image with RGB triplets below the bottom 1st percentile.
3.  These triplets serve as an overestimate of backscatter $B_s$.
4.  Use nonlinear least squares fitting on pairs $(B_s, z)$ to estimate $B_\infty$ and $B_b$.
5.  Re-estimate parameters every 500 optimization steps [S7].

**Depth Extraction:**
Depth $z$ is extracted from the current scene estimate at each pixel:
$$ z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i} $$
where $d_i$ is the depth of Gaussian $i$ [S7].

**Pitfall:** Camera sensors must have a linear response to light. In-camera processed .jpg or gamma-corrected images cannot be used unless non-linearities are reversed [S7].

**Parameter Bounds:** $B_\infty \in [0, 1]$, $B_b \in [0, 5]$ [S7].

### Variant: Multi-Scale Constraint
Improves rendering of distant objects by applying position-aware scale control [S8].

**Formula:**
Define a scale factor $\gamma(\mu)$ based on the distance of Gaussian center $\mu$ from camera center $c$ and scene radius $r$:
$$ \gamma(\mu) = \begin{cases} 1 & \text{if } \|\mu - c\|^2 < 2r \\ \|\mu - c\|^2 / r - 1 & \text{if } \|\mu - c\|^2 \ge 2r \end{cases} $$

**Operations:**
*   **Clone:** If $\max(s) < g \cdot \gamma(\mu)$, clone the Gaussian.
*   **Split:** If $\max(s) \ge g \cdot \gamma(\mu)$, split the Gaussian.
*   **Eliminate:** If $\max(s) > b \cdot \gamma(\mu)$, eliminate the Gaussian.
where $g$ and $b$ are scale threshold parameters [S8].

### Variant: GS-IR Representation
Extends 3DGS for inverse rendering by adding normal vectors and material properties [S9].

**Layout:**
*   **Geometric Properties ($\mathbf{T}$):** Position $\mathbf{p}$, scaling $\mathbf{s}$, rotation quaternion $\mathbf{q}$, and normal vector $\mathbf{n}$.
*   **Material Properties ($\mathbf{M}$):** Albedo $\mathbf{a}$, roughness $\rho$, and metallicity $m$.
*   **Appearance ($\mathbf{A}$):** Opacity $\alpha$ and SH coefficients $\mathbf{f}_c$ [S9].

**Training Stages:**
1.  **Initial Stage:** Optimize $\mathbf{T}$ and $\mathbf{A}$ using L1 color loss and normal loss for 30K iterations.
2.  **Decomposition Stage:** Fix $\mathbf{T}$ and $\mathbf{A}$, optimize $\mathbf{M}$ using decomposition loss for 10K iterations [S9].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. Ensures unbiased transmittance isosurfaces and prevents surface erosion [9624-continuous-attenuation-model-for-3d-gaussian-splatting.md].

### DAVIGS
A plug-and-play appearance modeling method that mitigates floaters and color distortions by transforming rendered images using 3D consistent features queried from multi-resolution hash grids. Maintains real-time rendering by discarding the appearance module after optimization [12021-davigs.md].

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses computational bottlenecks in sorting and attribute fetching. Maps software strategies to hardware constraints (Roofline model) or uses comparison-free logic and sorting-free approximations [12180-sorting-and-rendering-pipeline-optimization.md].

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes 3DGS scenes to incorporate semantic language features (e.g., CLIP). Disables densification and pruning during language-aware training to maintain semantic consistency [12438-langsplat-training-procedure.md].

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps. Uses dynamic texel allocation and splitting-based densification to avoid convergence issues associated with cloning textured primitives [12606-content-aware-texturing-for-gaussian-splatting.md].

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Defines the fundamental 3D Gaussian primitive and its lifecycle (truncation, pruning, atomization, voxelization). Focuses on the definition of the primitive rather than the rasterization kernel [12855-gaussian-representation-and-truncation.md].

### [DeSplat](12925-desplat.md)
Decomposes 3DGS into static scene Gaussians and per-view distractor Gaussians to remove transient occluders (e.g., pedestrians) without external semantic models [12925-desplat.md].

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Adapts 3DGS for 360-degree equirectangular images by modifying the rasterization module for spherical rendering and using OpenSfM instead of Colmap [12929-360-gaussian-splatting.md].

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint by compressing numerical representations, clustering parameters into codebooks, or replacing explicit attributes with neural fields [12943-data-compression-and-storage-optimization.md].

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of discontinuous indicator functions with respect to Bézier curve control points, enabling backpropagation in discontinuity-aware rendering [13137-b-zier-boundary-gradient-approximation.md].

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Enables 3DGS to render sharp boundaries by introducing "pre-scissoring" along cubic Bézier curves and using Bézier-boundary gradient approximation [13138-discontinuity-aware-gaussian-splatting.md].

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Uses covariance-based feature augmentation to capture co-variation patterns, allowing smaller feature vectors without sacrificing detail [13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md].

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Integrates ocean optics models (direct attenuation, backscatter) and uses alternating optimization to refine depth and medium parameters simultaneously [13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md].

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Densifies 2D Gaussians in image space and back-projects them to 3D to initialize view-dependent kernels for complex specular radiance [13985-error-driven-2d-gaussian-training.md].

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
Converts the volumetric 3DGS representation into explicit triangular meshes or point clouds for compatibility with standard 3D graphics pipelines [14095-mesh-extraction-and-representation.md].

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Reduces computational overhead of view-dependent appearance by compressing SH coefficients, reducing polynomial degree, or optimizing SH gradient handling [14206-spherical-harmonics-optimization.md].

### [TraGraph-GS](14454-tragraph-gs.md)
Uses graph-based adaptive spatial partitioning and camera connectivity graphs to handle large-scale scenes and eliminate boundary artifacts [14454-tragraph-gs.md].

### [Explicit 4D Gaussian Splatting](14584-explicit-4d-gaussian-splatting.md)
Separates Gaussians into static and dynamic categories, explicitly sampling positions and rotations at sparse timestamps to achieve memory efficiency in dynamic scenes [14584-explicit-4d-gaussian-splatting.md].

### [Color and Appearance Modeling](14740-color-and-appearance-modeling.md)
Extends radiance representation to handle complex lighting and view-dependent effects by decoupling geometry from lighting or using dynamic appearance embeddings [14740-color-and-appearance-modeling.md].

### [GS-IR](14852-gs-ir.md)
Leverages forward mapping volume rendering with material properties (albedo, roughness, metallicity) and normal vectors for photorealistic novel view synthesis and relighting [14852-gs-ir.md].

## Operating characteristics

**Time and Memory Complexity:**
*   **Memory:** Standard 3DGS memory usage scales linearly with the number of Gaussians. Large scenes can exceed 1 GB [12943-data-compression-and-storage-optimization.md]. Half-precision fetches reduce register pressure and memory bandwidth usage [S1].
*   **Time:** Rendering is real-time capable due to tile-based rasterization. Optimization time depends on the number of Gaussians and iterations.

**Measured Performance:**
*   In urban view synthesis (KITTI-360, Waymo), implementations using `gsplat` and sparse convolutions (e.g., EVolSplat) achieve high PSNR/SSIM with low memory consumption (e.g., 10.41 GB for Waymo) and high FPS (83.81 FPS) [S2].

**Strengths and Limitations:**
*   **Strengths:** Real-time rendering, high fidelity, differentiable optimization.
*   **Limitations:** High memory footprint for large scenes, sensitivity to camera distribution for scene extent, artifacts in sparse views or noisy data without variants like Denoise-GS [S5].

**When to Use:**
*   Use standard implementation for dense-view, well-lit scenes.
*   Use variants (Corrected Scene-Extent, Denoise-GS, Underwater) for specific challenges like clustered cameras, sparse/noisy views, or aquatic environments.

## Background

3D Gaussian Splatting was introduced as a real-time explicit scene representation method [26]. The implementation details described here are based on the `gsplat` library [S2] and various extensions addressing specific limitations such as scene extent bias [S3], defocus blur [S4], noise [S5], specular highlights [S6, unofficial], underwater scattering [S7], large-scale partitioning [S8], and inverse rendering [S9]. The use of half-precision kernels and specific loss scaling strategies [S1] are critical for stable training on modern GPUs.

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
