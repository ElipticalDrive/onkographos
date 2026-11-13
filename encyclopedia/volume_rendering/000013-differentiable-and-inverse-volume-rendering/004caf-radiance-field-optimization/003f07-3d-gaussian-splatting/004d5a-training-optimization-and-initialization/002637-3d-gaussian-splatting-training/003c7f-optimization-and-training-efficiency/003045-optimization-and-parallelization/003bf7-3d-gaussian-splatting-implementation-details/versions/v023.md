# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation configurations, optimization strategies, and variant methods required to train and render 3D Gaussian Splatting (3DGS) effectively. It addresses the core differentiable rasterization pipeline, including half-precision memory management, loss scaling, and specific hyperparameter settings for optimizers and scene extents. It also covers specialized variants for handling defocus blur, view-dependent color compression, sparse-view denoising, and underwater scattering, which modify the standard forward and backward passes to address specific scene conditions or computational bottlenecks.

## How it works
The standard 3DGS training pipeline relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters [S2].

### Standard Implementation Configuration
The following parameters and steps define the baseline implementation for large-scale or standard 3DGS training:

1.  **Rasterization Library**: Use `gsplat` as the Gaussian rasterization library [S2].
2.  **Optimization**: Use the Adam optimizer with a learning rate of $1 \times 10^{-3}$ [S2].
3.  **Spherical Harmonics (SH)**: Set the SH degree to 1 for simplicity and efficiency, unless higher fidelity is required [S2].
4.  **Training Loop**:
    *   Randomly select a single image from a random scene as supervision for each iteration [S2].
    *   Apply an automatic scaling factor to loss values to improve half-precision training stability [S1].
5.  **Memory Management**:
    *   Use half-precision (FP16) memory fetches on feature vectors during rasterization for both forward and backward kernels to reduce register pressure [S1].
    *   Use tiny-cuda-nn's JIT-compiled cooperative vector MLPs, trained and evaluated in half precision (FP16) if neural fields are used for appearance [S1].

### Corrected Scene Extent
The default 3DGS scene extent calculation is biased by camera distribution. If cameras are clustered (e.g., orbiting a small object), the calculated extent is small, leading to incorrect Gaussian splitting/pruning [S3].

**Formula for Corrected Scene Extent:**
$$e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2$$

Where:
*   $e'_{\text{scene}}$ is the corrected scene extent.
*   $N_{\text{SfM}}$ is the number of Structure-from-Motion (SfM) points.
*   $p_i$ is the $i$-th SfM point.
*   $\bar{C}$ is the average camera position: $\bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i$, where $N_{\text{cam}}$ is the number of camera views and $C_i$ is the position of the $i$-th camera [S3].

### Selective Defocus Blurring
To handle spatially varying blur, predict a per-Gaussian scale offset $\delta s_j$ using a compact MLP [S4].

**Steps:**
1.  Predict $\delta s_j$ for each Gaussian $j$.
2.  Add $\delta s_j$ to the base scale to transform the covariance matrix.
3.  Project the transformed covariance to 2D space.
4.  Use the projected covariance as a pixel-wise blur kernel during rasterization [S4].

**Pitfall:** If $\delta s_j$ is predicted as a constant for all Gaussians, the method degrades to a uniform Gaussian blur over the entire image [S4].

### Compact View-dependent Color
Replace per-Gaussian SH coefficients (48 parameters) with a shared grid-based neural field to reduce memory [S5].

**Steps:**
1.  Contract unbounded positions $p_n \in \mathbb{R}^3$ to a bounded range using the mip-NeRF 360 contraction function:
    $$\text{contract}(p_n) = \begin{cases} p_n & \text{if } \|p_n\| \le 1 \\ \frac{p_n}{\|p_n\|} (2 - \|p_n\|) & \text{if } \|p_n\| > 1 \end{cases}$$
2.  Input contracted positions into hash grids.
3.  Feed the resulting features and view direction $d$ into a tiny MLP $f(\cdot; \theta)$ to compute view-dependent color:
    $$c_n(d) = f(\text{contract}(p_n), d; \theta)$$
4.  Use 0-degree SH components (RGB channels) as input features for the MLP [S5].

### Denoise-GS (Sparse-View Training)
A two-round self-supervised denoising process for sparse-view scenarios [S6].

**Round 1:**
1.  Train a UNet using Noise2Void (N2V) for unsupervised denoising.
2.  Process denoised images through InstantSplat to initialize and train 3DGS.
3.  Render results and group/filter by viewpoint to ensure representativeness.

**Round 2:**
1.  Fine-tune the same UNet using a composite loss balancing low-frequency consistency and perceptual detail consistency.
2.  Render again on selected viewpoints to generate denoised views [S6].

### View-Dependent Opacity Kernel
Model specular reflectance lobes by making opacity view-dependent [S7, unofficial].

**Formula:**
$$\hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos(\max(0, \min(\frac{2\theta}{T}, \pi)))+1}{2}\exp(\beta)$$

Where:
*   $\alpha$: Base opacity.
*   $\theta$: Angle between view direction and lobe central orientation.
*   $T$: Angular span of the lobe (initialized as $c\bar{\theta}_i/\pi$).
*   $\beta$: Sharpness modulator (initialized to 0) [S7, unofficial].

**Pitfall:** As $\theta$ increases, transparency diminishes to zero, preventing artifacts outside the lobe's influence [S7, unofficial].

### Underwater Rendering (Backscatter Estimation)
Adapt 3DGS for underwater scenes by estimating direct attenuation ($B_d$), infinite backscatter ($B_\infty$), and backscatter coefficient ($B_b$) [S8].

**Backscatter Estimation Algorithm:**
1.  Divide the depth map (from current scene estimate) into 10 evenly-spaced clusters from $\min(z)$ to $\max(z)$.
2.  In each cluster, identify pixels in the ground-truth image with RGB triplets below the bottom 1st percentile.
3.  These triplets serve as an overestimate of backscatter $B_s$.
4.  Use nonlinear least squares fitting on pairs $(B_s, z)$ to estimate $B_\infty$ and $B_b$.
5.  Re-estimate parameters every 500 optimization steps [S8].

**Depth Extraction for Parameter Update:**
Extract depth $z$ from the current scene estimate at each pixel:
$$z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i}$$
Where $d_i$ is the depth of Gaussian $i$, $\alpha_i$ is opacity, and $T_i$ is transmittance [S8].

**Pitfall:** Camera sensor must have a linear response to light. In-camera processed .jpg or gamma-corrected images cannot be used unless non-linearities are reversed [S8].

**Loss Function:**
$$L = \sum_{k=1}^K (1 - \lambda_1) \| I_k^{gt} - I_k^r(\Theta) \|_{\ell_1} + \lambda_1 L_{D-SSIM}(I_k^{gt}, I_k^r(\Theta))$$
Where $K$ is the number of images and $\lambda_1 \in [0,1]$ [S8].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework for unbiased surface extraction.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
Derives analytical Jacobians for SE(3) camera pose to enable efficient gradient-based camera tracking in SLAM.

### [DAVIGS](12021-davigs.md)
Mitigates appearance variation-induced floaters by transforming rendered images using 3D consistent features.

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses computational bottlenecks in sorting and attribute fetching using hardware-friendly strategies.

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes scenes to incorporate semantic language features alongside photometric rendering.

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps for adaptive appearance.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
Provides a constant-time ray oracle for secondary rays (shadows, AO) without mesh extraction.

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Modifies core Gaussian representation and culling thresholds to improve fidelity and stability.

### [DeSplat](12925-desplat.md)
Decomposes scene into static and per-view distractor Gaussians for distractor-free novel view synthesis.

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Supports 360-degree equirectangular images by modifying the rasterization module and using OpenSfM.

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint via compression, clustering, or neural field replacements.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Places isometric Gaussians on a fixed voxel grid for CT volume extraction and X-ray rendering.

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of discontinuous indicator functions for boundary-aware rendering.

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Enables rendering of sharp boundaries using pre-scissoring and Bézier-boundary gradients.

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Uses second-order statistics to reduce anchor feature dimensions while maintaining quality.

### [3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md)
Models outgoing radiance as a function of local illumination and BRDF attributes.

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Replaces discrete opacity with Gaussian kernel functions for continuous surface reconstruction.

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Integrates ocean optics models to correct color distortion and visibility loss underwater.

### [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md)
Enables multi-view mini-batch training to reduce gradient variance and stabilize optimization.

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Initializes 3D Gaussians by optimizing 2D kernels in image space and back-projecting to 3D.

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
Techniques for extracting meshes from Gaussian splatting representations.

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Reduces computational cost of SH representation via distillation, quantization, or degree reduction.

### [SWAG (Splatting in the Wild images with Appearance-conditioned Gaussians)](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md)
Handles unstructured in-the-wild images by disentangling static geometry from dynamic appearance.

## Operating characteristics
*   **Memory**: Standard 3DGS with SH degree 3 requires 48 parameters per Gaussian for color alone. Using compact view-dependent color (hash grids + MLP) reduces this significantly [S5]. Half-precision rasterization reduces register pressure and memory bandwidth usage [S1].
*   **Speed**: The `gsplat` library and half-precision kernels enable real-time rendering speeds [S1, S2]. Multi-view training pipelines can further optimize gradient computation efficiency [S13982, unofficial].
*   **Stability**: Automatic loss scaling is required for half-precision training stability [S1]. Corrected scene extent prevents premature pruning or excessive splitting in scenes with clustered cameras [S3].
*   **Limitations**: Standard 3DGS assumes smooth alpha blending and perspective cameras. Variants are needed for discontinuities [S13138], 360-degree views [S12929, unofficial], and underwater scattering [S8]. Linear camera response is required for underwater backscatter estimation [S8].

## Background
3D Gaussian Splatting was introduced as a real-time explicit scene representation method [S26, referenced in S8]. The implementation details described here, such as the use of `gsplat` [S2, S5] and specific optimization strategies, have evolved to address the method's initial limitations in memory usage, handling of blur, and robustness in sparse or noisy views [S4, S6]. The correction of scene extent calculation addresses biases in the original adaptive density control mechanism [S3].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S5] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S6] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S7] Augmented Radiance Field: A General Framework for Enhanced Gaussian Splatting, https://doi.org/10.48550/arXiv.2602.19916, fetched 2026-10-30 (unofficial)
[S8] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
