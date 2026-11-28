# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and variant algorithms required to train and render 3D Gaussian Splatting (3DGS) scenes effectively. It addresses the gap between the theoretical differentiable rasterization framework and practical execution, focusing on memory management, precision handling, and specific architectural modifications for challenging environments (e.g., underwater, sparse views, defocus). It builds on [Optimization and Parallelization](6233-optimization-and-parallelization.md) by providing the concrete hyperparameters and kernel-level optimizations that enable stable convergence and high-fidelity rendering.

## How it works

### Standard Training Configuration
The baseline training process relies on the Adam optimizer with a learning rate of $1 \times 10^{-3}$ [S2]. The Gaussian Rasterization Library `gsplat` is typically used, which implements the 3DGUT formulation with custom CUDA kernels for forward and backward logic [S1, S2].

**Data Structures and Parameters:**
*   **Optimizer:** Adam [S2].
*   **Spherical Harmonics (SH) Degree:** Typically set to 1 for simplicity in large-scale or real-time applications, though degree 3 is standard for high fidelity [S2].
*   **Precision:** Half-precision (FP16) is used for memory fetches on feature vectors during rasterization (both forward and backward) to reduce register pressure [S1]. Neural network components (e.g., tiny MLPs for color) are trained and evaluated in FP16 using JIT-compiled cooperative vector MLPs (e.g., via `tiny-cuda-nn`) [S1].
*   **Loss Scaling:** An automatic scaling factor is applied to loss values to improve half-precision training stability [S1].
*   **Iteration Supervision:** During training, a single image is randomly selected from the scene as supervision for each iteration [S2].

### Scene Extent and Initialization
The scene extent $e_{\text{scene}}$ determines the spatial bounds for Gaussian densification and pruning.

**Default Calculation:**
The default 3DGS implementation calculates scene extent based on camera positions:
$$
e_{\text{scene}} = 1.1 \cdot \max_{i=1}^{N_{\text{cam}}} \|\bar{C} - C_i\|_2
$$
$$
\bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i
$$
where $N_{\text{cam}}$ is the number of camera views, $C_i$ is the position of the $i$-th camera, and $\bar{C}$ is the average camera position [S3].

**Pitfall:** This default formulation is biased if cameras are clustered (e.g., orbiting a small object in a large environment), leading to an underestimated scene extent and incorrect splitting/pruning of Gaussians [S3].

**Corrected Scene Extent:**
To avoid this bias, the scene extent can be calculated based on the Structure-from-Motion (SfM) point cloud:
$$
e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2
$$
where $N_{\text{SfM}}$ is the number of SfM points and $p_i$ is the $i$-th SfM point [S3].

### Variant: Selective Defocus Blurring
To handle spatially varying blur (e.g., defocus), a per-Gaussian scale offset $\delta s_j$ is predicted using a compact MLP [S4].

**Steps:**
1.  Predict $\delta s_j$ for each Gaussian $j$.
2.  Add $\delta s_j$ to the base scale to transform the covariance matrix.
3.  Project the transformed covariance to 2D space.
4.  Use the projected 2D covariance as a pixel-wise blur kernel during rasterization [S4].

**Pitfall:** If $\delta s_j$ is predicted as a constant for all Gaussians, the method degrades to a uniform Gaussian blur over the entire image, failing to model spatially varying blurriness [S4].

### Variant: Compact View-dependent Color
To reduce the parameter count from 48 SH coefficients per Gaussian to a shared grid-based neural field, the following procedure is used [S5]:

**Steps:**
1.  Contract unbounded positions $p_n \in \mathbb{R}^3$ to a bounded range using the mip-NeRF 360 contraction function:
    $$
    \text{contract}(p_n) = 
    \begin{cases} 
    p_n & \text{if } \|p_n\| \le 1 \\
    \frac{p_n}{\|p_n\|} (2 - \|p_n\|) & \text{if } \|p_n\| > 1
    \end{cases}
    $$
2.  Input the contracted positions into hash grids.
3.  Feed the resulting features and the view direction $d$ into a tiny MLP $f(\cdot; \theta)$ to compute view-dependent color:
    $$
    c_n(d) = f(\text{contract}(p_n), d; \theta)
    $$
4.  Use the 0-degree components of SH (RGB channels, non-view-dependent) as input features to the MLP, converting them to RGB colors [S5].

### Variant: Denoise-GS (Sparse/Noisy Views)
This method combines unsupervised image denoising with 3DGS training in two rounds [S6]:

**Round 1:**
1.  Train a UNet using Noise2Void (N2V) for denoising.
2.  Process denoised images through InstantSplat to initialize, train, and render 3DGS.
3.  Group and filter rendered results by viewpoint to ensure representativeness.

**Round 2:**
1.  Fine-tune the same UNet using a composite loss balancing low-frequency consistency and perceptual detail consistency.
2.  Render again on selected viewpoints to generate denoised views [S6].

**Comparison:** Unlike supervised methods or Noise2Noise (N2N), N2V requires no paired clean data, making it suitable for sparse viewpoints where only one image per view exists [S6].

### Variant: View-Dependent Opacity Kernel
To model specular reflectance, a transparent Gaussian kernel with view-dependent opacity is used [S7, unofficial]:

**Formula:**
$$
\hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos(\max(0, \min(\frac{2\theta}{T}, \pi)))+1}{2}\exp(\beta)
$$
where:
*   $\alpha$: Base opacity.
*   $\theta$: Angle between view direction and lobe central orientation.
*   $T$: Controls angular span (initialized as $c\bar{\theta}_i/\pi$ based on viewpoint density).
*   $\beta$: Modulates sharpness (initialized to zero).

**Pitfall:** As $\theta$ increases, transparency diminishes to zero, preventing artifacts outside the lobe's influence [S7, unofficial]. This adds 5 learnable parameters to the standard configuration [S7, unofficial].

### Variant: Underwater Rendering (Gaussian Splashing)
This method adapts 3DGS for underwater environments by learning direct attenuation ($B_d$), infinite backscatter ($B_\infty$), and backscatter coefficient ($B_b$) [S8].

**Rendering Formula:**
$$
C_{uw} = \sum_{i=1}^M c_i \alpha_i T_i e^{-B_d z} + B_\infty (1 - e^{-B_b z})
$$
where $T_i = \prod_{j=0}^{i-1} (1 - \alpha_j)$ and $z$ is the depth [S8].

**Backscatter Estimation Steps:**
1.  Divide the depth map (from current scene estimate) into 10 evenly-spaced clusters from $\min(z)$ to $\max(z)$.
2.  Within each cluster, identify pixels in the ground-truth image whose RGB triplets are below the bottom 1st percentile.
3.  These triplets serve as an overestimate of backscatter $B_s$.
4.  Use nonlinear least squares fitting on pairs $(B_s, z)$ to estimate $B_\infty$ and $B_b$.
5.  Re-estimate $B_\infty$ and $B_b$ every 500 optimization steps [S8].

**Depth Extraction:**
Depth $z$ is extracted from the current scene estimate at each pixel:
$$
z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i}
$$
where $d_i$ is the depth of Gaussian $i$ [S8].

**Pitfall:** The camera sensor must have a linear response to light. In-camera processed .jpg images or gamma-corrected images cannot be used unless non-linearities are reversed [S8].

**Parameter Bounds:** $B_\infty \in [0, 1]$, $B_b \in [0, 5]$ [S8].

### Variant: Gaussian Reduction with Learnable Mask
To reduce the number of Gaussians, a learnable mask $M$ is applied during optimization [S9]:

**Steps:**
1.  Apply binary masks $M$ to opacities and scale attributes at every densification step.
2.  Use residual vector quantization (R-VQ) with $L$ stages for geometric attributes (scale, rotation).
3.  Represent view-dependent colors via a grid-based neural field (hash grids + MLP) [S9].

**Masking Formulation:**
$$
M_n = \text{sg}(\mathbb{1}[\sigma(m_n) > \epsilon] - \sigma(m_n)) + \sigma(m_n)
$$
$$
\hat{s}_n = M_n s_n, \quad \hat{o}_n = M_n o_n
$$
where $\epsilon$ is the masking threshold, $\text{sg}(\cdot)$ is the stop gradient operator, $\mathbb{1}[\cdot]$ is the indicator function, and $\sigma(\cdot)$ is the sigmoid function [S9].

**Loss Function:**
$$
L = L_{\text{ren}} + \lambda_m L_m + L_r + L_s
$$
where $L_{\text{ren}}$ is L1+SSIM loss, $L_m = \frac{1}{N} \sum_{n=1}^N \sigma(m_n)$ is the masking loss, and $L_r, L_s$ are rotation and scale objective functions. $\lambda_m$ is typically $5 \times 10^{-4}$ [S9].

### Variant: 3DGS Visibility Pruning
This method prunes occluded Gaussians using their centroids as reference points [S9]:

**Steps:**
1.  Represent each Gaussian kernel by its centroid position, scaling matrix $S$, and rotation matrix $R$ (defining covariance $\Sigma$).
2.  Use centroids as 3D points $p_i$ in the visibility mask computation.
3.  For each viewpoint, project points to 2D, sort by depth, and mark points as visible if they fall within the top $\tau$ depth layers.
4.  Remove Gaussians marked as occluded during the Adaptive Density Control block [S9].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. Ensures unbiased, ellipsoidal transmittance isosurfaces and prevents surface erosion.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
Derives analytical Jacobians of SE(3) camera poses with respect to 3D Gaussians for efficient gradient-based camera tracking in SLAM, avoiding automatic differentiation overhead.

### [DAVIGS](12021-davigs.md)
A plug-and-play appearance modeling method that mitigates floaters and color distortions by transforming rendered images using 3D consistent features queried from multi-resolution hash grids.

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses computational bottlenecks in sorting and attribute fetching by mapping software strategies to hardware constraints (Roofline model) or using comparison-free sorting logic.

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes 3DGS scenes to incorporate semantic language features (e.g., CLIP) alongside photometric rendering, disabling densification/pruning during language-aware training.

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps and dynamic texel allocation to adapt appearance to captured content without pretrained models.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
A neural field mapping rays to distance and visibility for constant-time, constant-memory ray queries (shadows, GI) without mesh extraction or BVH traversal.

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Modifies core Gaussian representation, culling thresholds, and densification strategies to improve fidelity and stability by addressing aggressive opacity falloff and uncontrolled proliferation.

### [DeSplat](12925-desplat.md)
Decomposes 3DGS into static scene Gaussians and per-view distractor Gaussians to remove transient occluders (pedestrians, vehicles) using only volume rendering and photometric loss.

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Generates 3D representations from 360-degree equirectangular images by modifying the rasterizer for spherical rendering and using OpenSfM for initialization.

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint and storage overhead by compressing numerical representations, clustering parameters into codebooks, or replacing explicit attributes with neural fields.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Places isometric 3D Gaussians at fixed voxel grid centroids to enable efficient CT volume extraction and differentiable X-ray rendering, fixing positions and removing color coefficients.

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of discontinuous indicator functions with respect to Bézier curve control points to enable backpropagation in discontinuity-aware rendering.

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Enables 3DGS to render sharp boundaries by introducing "pre-scissoring" along cubic Bézier curves and using Bézier-boundary gradient approximation.

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Introduces second-order statistics via covariance-based feature augmentation to achieve superior quality with smaller anchor feature dimensions.

### [3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md)
Decouples scene illumination from material properties using a learnable factorised tensor grid for local incident illumination, enabling realistic specular effects.

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Establishes a continuous scene representation via kernel regression, replacing discrete opacity values with Gaussian kernel functions for direct mesh extraction.

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Adapts 3DGS to underwater environments by learning direct attenuation and backscatter parameters, using an alternating optimization strategy for depth and medium parameters.

### [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md)
Enables multi-view mini-batch training to reduce gradient variance, using a thread-efficient partial rendering pipeline and adapting Adaptive Density Control.

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Densifies 2D Gaussian kernels in image space using a differentiable rasterizer, then back-projects them to 3D to initialize new 3D Gaussians for complex specular radiance.

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
Techniques for extracting watertight meshes from 3D Gaussian Splatting representations, often leveraging the continuous fields or density estimates derived from the Gaussians.

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Reduces computational cost and memory footprint of view-dependent color by modifying SH representation through distillation, quantization, or degree reduction.

## Operating characteristics

**Time and Memory Complexity:**
Standard 3DGS training is memory-intensive due to the storage of per-Gaussian attributes (position, scale, rotation, opacity, SH coefficients). Large scenes can exceed 1 GB [S1, S12943]. The rasterization process is parallelized across pixels and splats, with complexity dependent on the number of Gaussians per pixel [S2].

**Measured Performance:**
*   **Standard 3DGS:** On an NVIDIA A6000 GPU, training on Mip-NeRF 360 takes ~19m 22s with 154 fps rendering [S5].
*   **Compact 3DGS:** Using grid-based neural fields for color, training time reduces to ~17m 51s with 254 fps rendering on the same hardware, with comparable PSNR (29.91 vs 29.87) [S5].
*   **EVolSplat:** On an RTX4090, achieves 83.81 FPS with 10.41 GB memory on Waymo dataset, outperforming MVSplat (70.46 FPS, 26.75 GB) [S2].

**Strengths and Limitations:**
*   **Strengths:** Real-time rendering speeds, high-fidelity novel view synthesis, explicit scene representation allowing for easy editing and compression [S1, S2].
*   **Limitations:** High memory consumption for large scenes, sensitivity to initialization (camera distribution bias), and difficulty with transparent/specular surfaces without extensions (e.g., View-Dependent Opacity) [S3][S7, unofficial].
*   **When to Use:** Real-time applications, high-quality novel view synthesis, scenes with complex geometry.
*   **When Not to Use:** Extremely large open-world scenes without compression/voxelization, scenes with severe transient occlusions without distractor separation, or underwater environments without specific backscatter modeling [S12943, S12925, S13980].

## Background

3D Gaussian Splatting was introduced as a real-time explicit scene representation method, building on the principles of differentiable rendering and volume rendering. It represents scenes as a set of 3D Gaussians, optimizing their parameters (position, covariance, opacity, color) to minimize rendering loss. The method gained popularity for its ability to achieve photorealistic results at interactive frame rates, surpassing NeRF-based methods in speed while maintaining high visual quality. Subsequent research has focused on optimizing training efficiency, reducing memory footprint, and extending the representation to handle specific challenges like defocus, underwater scattering, and semantic understanding.

## Sources

[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S5] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S6] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S7] Augmented Radiance Field: A General Framework for Enhanced Gaussian Splatting, https://doi.org/10.48550/arXiv.2602.19916, fetched 2026-10-30 (unofficial)
[S8] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
[S9] Visibility-Based Geometry Pruning of Neural Plenoptic Scene Representations, https://doi.org/10.1109/tmm.2025.3618548, fetched 2026-10-26
