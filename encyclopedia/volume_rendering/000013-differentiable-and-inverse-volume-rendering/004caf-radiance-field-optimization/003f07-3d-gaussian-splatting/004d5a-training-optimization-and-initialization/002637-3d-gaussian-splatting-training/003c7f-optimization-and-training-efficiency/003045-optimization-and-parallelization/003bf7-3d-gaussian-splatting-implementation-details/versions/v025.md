# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific software engineering choices, hyperparameters, and optimization variants required to implement 3D Gaussian Splatting (3DGS) effectively. It addresses the gap between the theoretical differentiable rendering formulation and a performant, stable training pipeline. Key implementation aspects include the use of half-precision arithmetic for memory efficiency, specific optimizer settings, and various methodological extensions for handling scene extent, defocus blur, view-dependent color, and underwater scattering.

## How it works
The implementation relies on a differentiable rasterizer that renders images from a set of 3D Gaussians. The core optimization mechanism involves computing the gradient of a loss function $\mathcal{L}$ (e.g., L1, SSIM) with respect to the Gaussian parameters (position, covariance, opacity, color, SH coefficients) and any auxiliary parameters.

### Standard Training Configuration
The baseline implementation uses the `gsplat` library [S2] with custom CUDA kernels for forward and backward logic [S1].

**Inputs and Outputs:**
*   **Input:** A set of images with known camera poses and an initial point cloud (e.g., from SfM).
*   **Output:** Optimized 3D Gaussian parameters (position $\mathbf{p}$, rotation $\mathbf{q}$, scale $\mathbf{s}$, opacity $\alpha$, SH coefficients $\mathbf{c}$).

**Data Structures:**
*   **Gaussian Attributes:** Stored in contiguous memory for efficient access.
*   **Feature Vectors:** During rasterization, half-precision memory fetches are used for feature vectors in both forward and backward kernels to reduce register pressure [S1].

**Optimization Steps:**
1.  **Initialization:** Initialize Gaussians from SfM points.
2.  **Iteration:**
    *   Randomly select a single image from the training set as supervision for each iteration [S2].
    *   Render the selected image using the differentiable rasterizer.
    *   Compute the loss $\mathcal{L}$ between the rendered image and the ground truth.
    *   Apply an automatic scaling factor to the loss values to improve half-precision training stability [S1].
    *   Compute gradients $\frac{\partial \mathcal{L}}{\partial \theta}$ via backpropagation.
    *   Update parameters using the Adam optimizer.

**Parameters:**
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** $1 \times 10^{-3}$ [S2].
*   **SH Degree:** 1 (for simplicity and performance) [S2].
*   **Precision:** Half-precision (FP16) for neural network evaluation and rasterization memory fetches [S1].

### Variants and Extensions
The following subsections describe specific implementation details for handling edge cases, scene characteristics, and performance optimizations.

#### Corrected Scene-Extent
The default 3DGS scene extent $e_{\text{scene}}$ is proportional to the distance of the furthest camera from the average camera position:
$$e_{\text{scene}} = 1.1 \cdot \max_{i=1}^{N_{\text{cam}}} \|\bar{C} - C_i\|_2$$
where $\bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i$. This formulation is biased if cameras are clustered (e.g., orbiting a small object in a large environment), leading to incorrect splitting/pruning [S3].

**Correction Formula:**
$$e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2$$
where $N_{\text{SfM}}$ is the number of Structure-from-Motion (SfM) points and $p_i$ is the $i$-th SfM point [S3].

#### Selective Defocus Blurring
To handle spatially varying blur, a compact MLP predicts a per-Gaussian scale offset $\delta s_j$. This offset is added to the base scale to transform the covariance, which is then projected to 2D space to act as a pixel-wise blur kernel during rasterization [S4].
*   **Pitfall:** If $\delta s_j$ is predicted as a constant for all Gaussians, the method degrades to a uniform Gaussian blur over the entire image [S4].

#### Compact View-dependent Color
To reduce the parameter count from 48 (SH degree 3) per Gaussian, a grid-based neural field can be used [S5].
1.  Contract unbounded positions $p_n \in \mathbb{R}^3$ to a bounded range:
    $$\text{contract}(p_n) = \begin{cases} p_n & \text{if } \|p_n\| \le 1 \\ \frac{p_n}{\|p_n\|} (2 - \|p_n\|) & \text{if } \|p_n\| > 1 \end{cases}$$
2.  Input positions into hash grids.
3.  Feed the resulting feature and view direction $d$ into a tiny MLP $f(\cdot; \theta)$ to compute color:
    $$c_n(d) = f(\text{contract}(p_n), d; \theta)$$
4.  Use 0-degree components of SH (RGB channels) as input features [S5].

#### Denoise-GS
For sparse-view or noisy scenes, a two-round self-supervised denoising process is used [S6]:
1.  **Round 1:** Train a UNet using Noise2Void (N2V) for denoising. Process denoised images through InstantSplat to initialize and train 3DGS. Group and filter rendered results by viewpoint.
2.  **Round 2:** Fine-tune the UNet using a composite loss (low-frequency and perceptual detail consistency). Render again on selected viewpoints to generate denoised views [S6].

#### View-Dependent Opacity Kernel
To model specular reflectance, opacity is made view-dependent using a half-period cosine function [S7, unofficial]:
$$\hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos(\max(0, \min(\frac{2\theta}{T}, \pi)))+1}{2}\exp(\beta)$$
where $\alpha$ is base opacity, $\theta$ is the angle between view direction and lobe orientation, $T$ controls angular span, and $\beta$ modulates sharpness. This adds 5 learnable parameters [S7, unofficial].
*   **Pitfall:** If $\theta$ increases, transparency diminishes to zero, preventing artifacts outside the lobe's influence [S7, unofficial].

#### Backscatter Estimation and Optimization (Underwater)
For underwater scenes, parameters $B_\infty$ (infinite backscatter) and $B_b$ (backscatter coefficient) are learned [S8].
1.  **Estimation:** Divide the depth map into 10 clusters. Identify pixels in the ground-truth image with RGB triplets below the bottom 1st percentile in each cluster. These serve as an overestimate of backscatter $B_s$.
2.  **Fitting:** Use nonlinear least squares fitting on pairs $(B_s, z)$ to estimate $B_\infty$ and $B_b$.
3.  **Update:** Re-estimate parameters every 500 optimization steps [S8].
*   **Pitfall:** Camera sensor must have a linear response. In-camera processed .jpg or gamma-corrected images cannot be used unless non-linearities are reversed [S8].
*   **Depth Extraction:** Depth $z$ is extracted from the current scene estimate during rendering:
    $$z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i}$$
    where $d_i$ is the depth of Gaussian $i$ [S8].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework for unbiased surface extraction.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
Derives analytical Jacobians of SE(3) camera pose for efficient gradient-based camera tracking in SLAM.

### [DAVIGS](12021-davigs.md)
Mitigates appearance variation-induced floaters by transforming rendered images at the image level using 3D consistent features.

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses computational bottlenecks in sorting and rasterization by mapping software strategies to hardware constraints.

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes scenes to incorporate semantic language features alongside photometric rendering.

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussian primitives with spatially varying texture maps.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
Provides a constant-time, constant-memory ray oracle for secondary rays.

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Modifies spatial extent, density, and survival criteria of 3D Gaussian primitives.

### [DeSplat](12925-desplat.md)
Decomposes the scene into static and per-view distractor Gaussians for distractor-free novel view synthesis.

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Generates 3D representations from 360-degree camera images by modifying the rasterization module for spherical rendering.

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint by compressing numerical representations or clustering parameters.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Places isometric 3D Gaussians at fixed voxel centroids for CT volume extraction.

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of discontinuous indicator functions for boundary-aware rendering.

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Enables rendering of sharp boundaries and discontinuities using pre-scissoring.

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Introduces second-order statistics to reduce anchor feature dimensions.

### [3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md)
Models outgoing radiance as a function of local illumination and BRDF attributes.

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Establishes a continuous scene representation via kernel regression for surface reconstruction.

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Adapts 3DGS to underwater environments by learning direct attenuation and backscatter parameters.

### [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md)
Enables multi-view mini-batch training to reduce gradient variance.

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Initializes 3D Gaussians by optimizing 2D Gaussian kernels in image space.

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
Methods for extracting meshes from Gaussian splatting representations.

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Reduces computational cost of directional radiance representation.

### [SWAG (Splatting in the Wild images with Appearance-conditioned Gaussians)](14315-swag-splatting-in-the-wild-images-with-appearance-conditione.md)
Handles unstructured image collections by disentangling static geometry from dynamic appearance.

## Operating characteristics
*   **Memory:** Standard 3DGS with SH degree 3 requires 48 parameters per Gaussian for color. Using compact view-dependent color (hash grids + MLP) reduces this significantly [S5]. Half-precision rasterization reduces register pressure [S1].
*   **Performance:** The use of `gsplat` and custom CUDA kernels enables real-time rendering. Training stability is improved by automatic loss scaling in FP16 [S1].
*   **Scalability:** Scene extent calculation based on SfM points (Corrected Scene-Extent) prevents bias in large outdoor scenes with clustered cameras [S3].
*   **Limitations:** Standard 3DGS assumes smooth alpha blending; discontinuities require specialized handling (e.g., DisC-GS) [S13138]. Underwater scenes require additional parameters and linear camera response [S8].

## Background
3D Gaussian Splatting was introduced as a real-time explicit scene representation method [S26, cited in S8]. The implementation details described here are based on the `gsplat` library [S2, S75] and various extensions addressing specific challenges such as scene extent bias [S3], defocus blur [S4], and underwater scattering [S8].

## Sources
[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S5] Compact 3D Gaussian Representation for Radiance Field, https://doi.org/10.1109/cvpr52733.2024.02052, fetched 2026-10-26
[S6] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S7] Augmented Radiance Field: A General Framework for Enhanced Gaussian Splatting, https://doi.org/10.48550/arXiv.2602.19916, fetched 2026-10-30 (unofficial)
[S8] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
