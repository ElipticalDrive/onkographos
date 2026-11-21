# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and variant algorithms required to train and render 3D Gaussian Splatting (3DGS) scenes effectively. It addresses the core differentiable rasterization pipeline, including half-precision memory management, loss scaling, and specific hyperparameter configurations for optimizers and scene extents. It also covers specialized variants for handling defocus blur, view-dependent opacity, underwater scattering, and memory compression.

## How it works

### Standard Training Configuration
The standard training pipeline relies on the `gsplat` library [S2] and custom CUDA kernels based on the 3DGUT formulation [S1].

**Inputs and Outputs:**
*   **Input:** A set of images with known camera poses, initialized 3D Gaussian primitives (position, covariance, opacity, color coefficients).
*   **Output:** Rendered images and gradients for Gaussian parameters.

**Data Structures:**
*   **Gaussian Attributes:** Position ($\mathbf{p}$), Rotation ($\mathbf{q}$), Scale ($\mathbf{s}$), Opacity ($\alpha$), Spherical Harmonics (SH) coefficients.
*   **Memory Layout:** Feature vectors are fetched in half-precision (FP16) to reduce register pressure during rasterization [S1].

**Optimization Steps:**
1.  **Initialization:** Initialize Gaussians from Structure-from-Motion (SfM) points.
2.  **Iteration:**
    *   Randomly select a single image from the training set as supervision [S2].
    *   Render the image using the differentiable rasterizer.
    *   Compute loss $\mathcal{L}$ (typically L1 + SSIM).
    *   Apply an automatic scaling factor to loss values to improve half-precision training stability [S1].
    *   Update Gaussian parameters using the Adam optimizer.

**Parameters:**
*   **Optimizer:** Adam with learning rate $1 \times 10^{-3}$ [S2].
*   **SH Degree:** Typically 1 for simplicity in large-scale scenes [S2], though standard 3DGS uses up to degree 3 (45 coefficients per RGB channel) [S5].
*   **Loss Scaling:** Automatic scaling factor applied to $\mathcal{L}$ [S1].

**Pitfalls:**
*   **Half-Precision Instability:** Without loss scaling, FP16 training may suffer from gradient underflow or overflow [S1].
*   **Camera Bias:** Default scene extent calculation based on camera positions can bias densification if cameras are clustered [S3].

### Corrected Scene Extent
The default 3DGS scene extent $e_{\text{scene}}$ is proportional to the distance of the furthest camera from the average camera position [S3]:
$$
e_{\text{scene}} = 1.1 \cdot \max_{i=1}^{N_{\text{cam}}} \|\bar{C} - C_i\|_2
$$
where $\bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i$ [S3]. This causes bias if cameras orbit a small object in a large environment [S3].

**Correction Formula:**
Use the SfM point cloud to calculate the corrected scene extent $e'_{\text{scene}}$:
$$
e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2
$$
where $N_{\text{SfM}}$ is the number of SfM points and $p_i$ is the $i$-th SfM point [S3].

### Selective Defocus Blurring
To handle spatially varying blur, predict a per-Gaussian scale offset $\delta s_j$ using a compact MLP [S4].

**Steps:**
1.  Predict $\delta s_j$ for each Gaussian $j$.
2.  Add $\delta s_j$ to the base scale to transform the covariance.
3.  Project the transformed covariance to 2D space.
4.  Use the projected covariance as a pixel-wise blur kernel during rasterization [S4].

**Pitfalls:**
*   If $\delta s_j$ is constant for all Gaussians, the method degrades to a uniform Gaussian blur [S4].

### Compact View-dependent Color
Replace per-Gaussian SH coefficients with a grid-based neural field to reduce parameters from 48 to a shared grid [S5].

**Steps:**
1.  Contract unbounded positions $p_n \in \mathbb{R}^3$ to a bounded range:
    $$
    \text{contract}(p_n) = \begin{cases} p_n & \text{if } \|p_n\| \le 1 \\ \frac{p_n}{\|p_n\|} (2 - \|p_n\|) & \text{if } \|p_n\| > 1 \end{cases}
    $$
2.  Input contracted positions into hash grids.
3.  Feed resulting features and view direction $d$ into a tiny MLP $f(\cdot; \theta)$ [S5].
4.  Compute view-dependent color: $c_n(d) = f(\text{contract}(p_n), d; \theta)$ [S5].
5.  Use 0-degree SH components (RGB channels, non-view-dependent) as input features [S5].

### View-Dependent Opacity Kernel
Model opacity as view-dependent using a half-period cosine function to approximate specular reflectance lobes [S7, unofficial].

**Formula:**
$$
\hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos(\max(0, \min(\frac{2\theta}{T}, \pi)))+1}{2}\exp(\beta)
$$
where $\alpha$ is base opacity, $\theta$ is the angle between view direction and lobe orientation, $T$ controls angular span, and $\beta$ modulates sharpness [S7, unofficial].

**Parameters:**
*   $T$: Initialized as $c\bar{\theta}_i/\pi$ based on viewpoint density [S7, unofficial].
*   $\beta$: Initialized to zero [S7, unofficial].

**Pitfalls:**
*   If $\theta$ increases, transparency diminishes to zero, preventing artifacts outside the lobe [S7, unofficial].

### Underwater Rendering (Gaussian Splashing)
Adapt 3DGS for underwater scenes by learning direct attenuation $B_d$, infinite backscatter $B_\infty$, and backscatter coefficient $B_b$ [S8].

**Rendering Formula:**
$$
C_{uw} = \sum_{i=1}^M c_i \alpha_i T_i e^{-B_d z} + B_\infty (1 - e^{-B_b z})
$$
where $T_i = \prod_{j=0}^{i-1} (1 - \alpha_j)$ [S8].

**Backscatter Estimation Steps:**
1.  Divide the depth map (from current scene estimate) into 10 evenly-spaced clusters [S8].
2.  Identify pixels in the ground-truth image with RGB triplets below the bottom 1st percentile within each cluster [S8].
3.  Use these triplets as an overestimate of backscatter $B_s$ [S8].
4.  Fit parameters $B_\infty$ and $B_b$ using nonlinear least squares on pairs $(B_s, z)$ [S8].
5.  Re-estimate backscatter parameters every 500 optimization steps [S8].

**Pitfalls:**
*   Camera sensor must have a linear response to light. In-camera processed .jpg or gamma-corrected images cannot be used unless non-linearities are reversed [S8].

**Depth Extraction:**
Extract depth $z$ from 3DGS for backscatter estimation:
$$
z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i}
$$
where $d_i$ is the depth of Gaussian $i$ [S8].

### Gaussian Reduction with Learnable Mask
Reduce Gaussian count by applying binary masks $M$ to opacities and scales during densification [S9].

**Steps:**
1.  Apply mask $M_n$ to scale $\hat{s}_n$ and opacity $\hat{o}_n$:
    $$
    M_n = \text{sg}(1[\sigma(m_n) > \epsilon] - \sigma(m_n)) + \sigma(m_n)
    $$
    $$
    \hat{s}_n = M_n s_n, \quad \hat{o}_n = M_n o_n
    $$
    where $\epsilon$ is the masking threshold, $\text{sg}(\cdot)$ is stop gradient, $1[\cdot]$ is indicator, and $\sigma(\cdot)$ is sigmoid [S9].
2.  Use residual vector quantization (R-VQ) with $L$ stages for geometric attributes (scale, rotation) [S9].
3.  Represent view-dependent colors via a grid-based neural field (hash grids + MLP) [S9].

**Loss Function:**
$$
L = L_{ren} + \lambda_m L_m + L_r + L_s
$$
where $L_{ren}$ is L1+SSIM loss, $L_m = \frac{1}{N} \sum_{n=1}^N \sigma(m_n)$ is masking loss, and $L_r, L_s$ are rotation and scale objective functions [S9].

**Parameters:**
*   $\lambda_m$: Weight for masking loss, typically $5 \times 10^{-4}$ [S9].

### 3DGS Visibility Pruning
Remove occluded Gaussians using visibility masks based on Gaussian centroids [S9].

**Steps:**
1.  Represent each Gaussian by its centroid position, scaling matrix $S$, and rotation matrix $R$ defining covariance $\Sigma$ [S9].
2.  Use centroids as 3D points $p_i$ in the visibility mask computation [S9].
3.  Compute visibility mask by projecting points to 2D and sorting by depth [S9].
4.  Remove Gaussians with zero visibility mask value during densification steps [S9].

### Denoise-GS (Sparse-View Training)
Combine unsupervised image denoising with 3DGS for sparse-view scenarios [S6].

**Steps:**
1.  **Round 1:** Train a UNet using Noise2Void (N2V) for denoising [S6].
2.  **Round 1:** Process denoised images through InstantSplat to initialize, train, and render 3DGS [S6].
3.  **Round 1:** Group and filter rendered results by viewpoint to ensure representativeness [S6].
4.  **Round 2:** Fine-tune the UNet using a composite loss balancing low-frequency consistency and perceptual detail consistency [S6].
5.  **Round 2:** Render again on selected viewpoints to generate denoised views [S6].

**Comparison:**
*   Unlike supervised methods, Denoise-GS uses unsupervised N2V, requiring no paired clean data [S6].
*   Unlike Noise2Noise (N2N), N2V requires no paired noisy images, suitable for sparse viewpoints [S6].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. Ensures unbiased transmittance isosurfaces and prevents surface erosion [9624-continuous-attenuation-model-for-3d-gaussian-splatting.md].

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
Derives analytical Jacobians of SE(3) camera pose with respect to 3D Gaussians for efficient gradient-based camera tracking in SLAM, avoiding automatic differentiation overhead [10077-explicit-camera-jacobian-derivation.md].

### DAVIGS
A plug-and-play appearance modeling method that mitigates floaters and color distortions by transforming rendered images at the image level using 3D consistent features, decoupling global view-dependent variations from local spatial changes [12021-davigs.md].

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses computational bottlenecks in 3DGS rendering by mapping software strategies to hardware constraints (Roofline model) or replacing standard sorting with hardware-friendly logic [12180-sorting-and-rendering-pipeline-optimization.md].

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes 3DGS scenes to incorporate semantic language features (e.g., CLIP) alongside photometric rendering, disabling densification/pruning during language-aware training [12438-langsplat-training-procedure.md].

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps and dynamic texel allocation to adapt appearance to captured content without pretrained models [12606-content-aware-texturing-for-gaussian-splatting.md].

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
A neural field mapping rays to distance and visibility for constant-time ray queries (shadows, AO) without mesh extraction or BVH traversal [12854-directed-distance-field-ddf-for-gaussian-splatting.md].

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Modifies core Gaussian representation, culling thresholds, and densification strategies to improve fidelity and stability, addressing opacity falloff and uncontrolled proliferation [12855-gaussian-representation-and-truncation.md].

### [DeSplat](12925-desplat.md)
Decomposes 3DGS into static scene Gaussians and per-view distractor Gaussians to remove transient occluders without external semantic models [12925-desplat.md].

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Modifies the differentiable rasterizer to support spherical rendering and uses OpenSfM for equirectangular inputs, solving the perspective camera limitation for 360-degree images [12929-360-gaussian-splatting.md].

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces memory footprint by compressing numerical representations, clustering parameters into codebooks, or replacing explicit attributes with neural fields [12943-data-compression-and-storage-optimization.md].

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Places isometric 3D Gaussians at fixed voxel grid centroids for efficient CT volume extraction and differentiable X-ray rendering, fixing positions and removing color coefficients [13071-voxel-based-gaussian-splatting.md].

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of discontinuous indicator functions with respect to Bézier curve control points, enabling backpropagation in discontinuity-aware rendering [13137-b-zier-boundary-gradient-approximation.md].

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Introduces "pre-scissoring" along cubic Bézier curves and uses Bézier-boundary gradient approximation to render sharp boundaries and discontinuities [13138-discontinuity-aware-gaussian-splatting.md].

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Uses covariance-based feature augmentation to capture co-variation patterns, reducing anchor feature dimensions while maintaining quality [13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md].

### [3iGS (Factorised Tensorial Illumination for 3D Gaussian Splatting)](13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md)
Models outgoing radiance as a function of continuous local illumination and individual Gaussian BRDF attributes, enabling accurate specular effects [13977-3igs-factorised-tensorial-illumination-for-3d-gaussian-splat.md].

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Replaces discrete opacity values with Gaussian kernel functions for continuous scene representation, enabling direct mesh extraction and reduced memory consumption [13979-gaussian-voxel-kernel-functions-gvkf.md].

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Integrates ocean optics models (direct attenuation, backscatter) and uses alternating optimization to refine depth and medium parameters simultaneously [13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md].

### [Efficient Multi-View Training for 3D Gaussian Splatting](13982-efficient-multi-view-training-for-3d-gaussian-splatting.md)
Enables multi-view mini-batch training with a thread-efficient partial rendering pipeline and adapted Adaptive Density Control to reduce gradient variance [13982-efficient-multi-view-training-for-3d-gaussian-splatting.md].

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Samples and optimizes 2D Gaussian kernels in image space, then back-projects to 3D for initialization, solving view-dependent kernel initialization for specular radiance [13985-error-driven-2d-gaussian-training.md].

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
Methods for extracting meshes from 3DGS representations [14095-mesh-extraction-and-representation.md].

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Reduces computational cost and memory footprint of SH representation through distillation, quantization, or degree reduction [14206-spherical-harmonics-optimization.md].

## Operating characteristics

**Time and Memory Complexity:**
*   Standard 3DGS uses $O(N)$ memory for $N$ Gaussians, with rendering complexity dependent on the number of splats per pixel [S2].
*   Compact methods (e.g., Compact View-dependent Color) reduce per-Gaussian parameters from 48 (SH degree 3) to a shared grid, significantly lowering memory [S5].
*   Gaussian Reduction with Learnable Mask reduces the number of Gaussians, lowering both memory and rendering time [S9].

**Measured Performance:**
*   On Mip-NeRF 360, Compact 3DGS achieved PSNR 29.91 with 17m 51s training time and 254 fps, compared to 3DGS (PSNR 29.87, 19m 22s, 154 fps) [S5].
*   On KITTI-360, EVolSplat achieved PSNR 23.26, SSIM 0.797, LPIPS 0.179, and 83.81 FPS with 10.41 GB memory [S2].

**Strengths and Limitations:**
*   **Strengths:** Real-time rendering speeds, high-fidelity reconstruction, differentiable optimization [S2, S5].
*   **Limitations:** High memory footprint for large scenes, sensitivity to camera distribution for scene extent, difficulty with transparent/specular surfaces without extensions [S3][S7, unofficial].

**When to Use:**
*   Use standard 3DGS for general-purpose novel view synthesis with dense views [S2].
*   Use Corrected Scene Extent for scenes with clustered cameras [S3].
*   Use Compact View-dependent Color for memory-constrained environments [S5].
*   Use Underwater Rendering for aquatic scenes [S8].
*   Use Denoise-GS for sparse-view or noisy scenarios [S6].

## Background

3D Gaussian Splatting was introduced as a real-time differentiable rendering method using 3D Gaussian primitives [S2]. The implementation details described here are based on the `gsplat` library [S2] and custom CUDA kernels [S1]. Variants have been developed to address specific challenges such as defocus blur [S4], view-dependent opacity [S7, unofficial], underwater scattering [S8], and memory compression [S5, S9].

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
