# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and variant configurations required to train and render 3D Gaussian Splatting (3DGS) scenes efficiently. It addresses common pitfalls in default configurations (such as scene extent bias and superfluous SH operations) and provides concrete hyperparameters for training stability. It serves as a practical supplement to the core [3D Gaussian Splatting](9128-3d-gaussian-splatting-3d-gaussian-splatting.md) method, focusing on the "how" of the training loop and rasterization engine rather than the theoretical derivation.

## How it works

### Standard Training Configuration
The baseline implementation relies on the `gsplat` library for differentiable rasterization [S2]. The training loop optimizes Gaussian parameters (position, covariance, opacity, color, SH coefficients) using the Adam optimizer [S2].

**Parameters:**
*   **Optimizer:** Adam [S2].
*   **Learning Rate:** $1 \times 10^{-3}$ [S2].
*   **Spherical Harmonics (SH) Degree:** Default is 3, but often set to 1 or 0 for efficiency [S2][S5, unofficial].
*   **Background Color:** Default is white (RGB 1.0). Changing to gray (e.g., 0.4) can improve training speed by ~21.85% without significant quality loss [S5, unofficial]. Optimal range is $0.3 \le \text{bg} \le 0.5$ [S5, unofficial].

**Implementation Steps:**
1.  **Initialization:** Initialize Gaussians from Structure-from-Motion (SfM) points.
2.  **Scene Extent Calculation:** Calculate the bounding volume for densification. The default calculation uses camera positions, which is biased if cameras are clustered [S3].
    *   Default: $e_{\text{scene}} = 1.1 \cdot \max_{i=1}^{N_{\text{cam}}} \|\bar{C} - C_i\|_2$ [S3].
    *   Corrected: $e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2$ [S3].
3.  **Rasterization:** Render images using the forward pass. To reduce register pressure, use half-precision (FP16) memory fetches for feature vectors in both forward and backward CUDA kernels [S1].
4.  **Loss Computation:** Compute photometric loss (L1/SSIM). Apply an automatic scaling factor to loss values to improve half-precision training stability [S1].
5.  **Optimization:** Update parameters using gradients. Randomly select a single image from a random scene as supervision for each iteration in some large-scale setups [S2].

### Optimization Strategies
Several modifications to the standard training pipeline improve speed or quality:

**1. SH Degree Zero Optimization**
In the original 3DGS code, SH coefficients with degree > 0 are often unchanged during training, making them superfluous [S5, unofficial].
*   **Action:** Set `self.sh_degree` to 0 and delete superfluous operations in the model code [S5, unofficial].
*   **Result:** Yields a 16.25% improvement in training speed; with code deletion, 16.57% improvement [S5, unofficial].

**2. Corrected Scene Extent**
To avoid bias from camera distribution (e.g., orbiting a small object), use the SfM point cloud to determine scene extent [S3].
*   **Formula:** $e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2$, where $\bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i$ [S3].

**3. Selective Defocus Blurring**
To handle motion or defocus blur, predict a per-Gaussian scale offset $\delta s_j$ using a compact MLP [S6].
*   **Step:** Add $\delta s_j$ to the base scale to transform the covariance. Project the transformed covariance to 2D space to act as a pixel-wise blur kernel [S6].
*   **Pitfall:** If $\delta s_j$ is constant, it degrades to a uniform Gaussian blur [S6].

**4. Compact View-dependent Color**
Replace per-Gaussian SH coefficients (48 params) with a grid-based neural field [S9].
*   **Contraction:** Contract unbounded positions $p_n$ to a bounded range:
    $$
    \text{contract}(p_n) = \begin{cases} 
    p_n & \text{if } \|p_n\| \le 1 \\
    \frac{p_n}{\|p_n\|} (2 - \|p_n\|) & \text{if } \|p_n\| > 1 
    \end{cases}
    $$
*   **Color:** $c_n(d) = f(\text{contract}(p_n), d; \theta)$, where $f$ is a tiny MLP [S9]. Use 0-degree SH components for base RGB [S9].

**5. Denoise-GS (Sparse View)**
For sparse/noisy views, combine unsupervised denoising with 3DGS [S11].
*   **Round 1:** Train a UNet using Noise2Void (N2V) on input images. Process denoised images through InstantSplat to initialize/train 3DGS.
*   **Round 2:** Fine-tune the UNet with a composite loss (low-frequency + perceptual). Render again on selected viewpoints [S11].

**6. SuGaR Mesh Extraction**
Extract a surface mesh by regularizing Gaussians to align with the scene surface [S8].
*   **Step:** Sample points on the visible level set of the Gaussian density function and apply Poisson reconstruction [S8].
*   **Parameter:** Octree Depth $D$ is critical; auto-tune $D$ based on scene complexity to reduce artifacts [S8].

**7. AtomGS (Geometry-Guided)**
Constrain Gaussians into "Atom Gaussians" (isotropic spheroids) for precise geometry [S10, unofficial].
*   **Step:** Apply progressive split schedule. Integrate 2D edge detector weights into curvature map for Edge-Aware Normal Loss [S10, unofficial].
*   **Pruning:** Merge Gaussians on large, less detailed surfaces [S10, unofficial].

**8. Triangle Soup (Mesh-based)**
Connect Gaussians to a mesh structure for editing [S7].
*   **Step:** Train GS with flat Gaussians. Construct/adjust Triangle Soup. For mesh input, fit pre-existing mesh (e.g., FLAME) [S7].
*   **Pitfall:** Artifacts appear with large faces; subdivide large faces [S7].

**9. Alias-free 4D Gaussian Splatting (Dynamic Scenes)**
For dynamic scenes, optimize static Gaussians for 3000 iterations as a warm-up, then jointly optimize Gaussians and deformation field [S4, unofficial].
*   **Loss:** $L = L_{\text{color}} + \lambda_1 L_{\text{scales}}$, with $\lambda_1 = 0.1$ [S4, unofficial].

## Subtopics

### [First-order Spherical Harmonics Distillation](9467-first-order-spherical-harmonics-distillation.md)
First-order Spherical Harmonics (SH) Distillation reduces the parameter count and computational cost of 3D Gaussian Splatting by distilling high-fidelity third-order SH coefficients from a pretrained teacher model into a compact first-order representation for a student model. Unlike standard degree reduction which simply truncates coefficients, this method uses knowledge distillation to preserve rendering quality despite the aggressive reduction from 16 coefficients per color channel to 4. It sets itself apart from sibling methods like [RoofGS](9830-roofgs.md) or [Inverse Multiquadric Splatting](9910-inverse-multiquadric-splatting.md) by specifically targeting the appearance representation's dimensionality rather than the geometric or sorting pipeline.

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
The Continuous Attenuation Model reformulates the rasterization-based image formation of 3D Gaussian Splatting into a volumetric ray-marching framework by mapping Gaussian primitives to a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. This equivalence allows the application of theoretical results from Other-at-Volume (OaV) for surface extraction, ensuring that transmittance isosurfaces remain unbiased and ellipsoidal. It distinguishes itself from prior attenuation approximations like those in GGGS [57] by enforcing geometric constraints that prevent surface erosion and multi-view inconsistency.

### [Explicit Camera Jacobian Derivation](10077-explicit-camera-jacobian-derivation.md)
This method derives analytical Jacobians of the SE(3) camera pose with respect to 3D Gaussian primitives to enable efficient gradient-based camera tracking in SLAM systems. It solves the problem of high computational overhead associated with automatic differentiation during rasterization, ensuring minimal parameterization using Lie algebra. Unlike prior ray-intersection methods that assume fewer Gaussians, this approach is applicable to the EWA splatting and 3D Gaussian Splatting (3DGS) rendering pipeline.

### DAVIGS
DAVIGS is a plug-and-play appearance modeling method for 3D Gaussian Splatting that mitigates appearance variation-induced floaters and color distortions by transforming rendered images at the image level using 3D consistent features [S1]. Unlike coupled methods that manipulate Gaussian properties (slowing rendering) or decoupled methods that only account for mild global variations, DAVIGS decouples global view-dependent appearance variations (e.g., daylight, ISPs) from local spatial changes (e.g., street lamps) by querying multi-resolution hash grids based on back-projected 3D coordinates [S1]. It builds 3D consistency across views implicitly while maintaining real-time rendering speeds by discarding the appearance module after optimization [S1].

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Sorting and Rendering Pipeline Optimization addresses the computational bottlenecks in 3D Gaussian Splatting (3DGS) rendering, specifically the high memory bandwidth requirements of depth sorting and the compute intensity of alpha blending. It sets itself apart from generic optimization by explicitly aligning software strategies with hardware bottlenecks identified via the Roofline model, applying memory-centric optimizations to memory-bound stages and compute-centric optimizations to compute-bound stages.

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
LangSplat Training Procedure is a method for optimizing 3D Gaussian Splatting scenes to incorporate semantic language features alongside photometric rendering. It extends the standard differentiable rendering pipeline by computing a loss against ground-truth language embeddings (e.g., CLIP features) when enabled, allowing the scene to support semantic queries. It is distinguished from standard 3D Gaussian Splatting training by the conditional inclusion of language feature loss and the consequent disabling of densification and pruning operations during language-aware training.

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
This method augments 2D Gaussian primitives with spatially varying texture maps that add offsets to spherical harmonic colors, allowing appearance to adapt to captured content without pretrained models. It distinguishes itself from sibling methods by using dynamic, content-aware texel allocation and splitting-based densification to manage geometric and appearance errors, avoiding the convergence issues associated with cloning textured primitives.

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
The Directed Distance Field (DDF) is a neural field that maps a ray (defined by origin $\mathbf{x}$ and direction $\mathbf{d}$) to the distance $t$ to the first surface along $\mathbf{d}$ and a visibility value $\xi$. It serves as a constant-time, constant-memory ray oracle for secondary rays (shadows, ambient occlusion, global illumination) in 3D Gaussian Splatting scenes, eliminating the need for mesh extraction or BVH traversal. Unlike BVH-based Gaussian tracing, which scales linearly with scene size, or sphere-traced Signed Distance Fields (SDFs), which require iterative evaluations, the DDF answers any ray query in a single forward pass with fixed memory usage.

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
This family defines the mathematical primitives for evaluating 3D Gaussian splats during rasterization and the mechanisms for culling fragments that contribute negligibly to the final image. It addresses the problem of balancing rendering fidelity with computational efficiency by determining how a Gaussian's spatial extent and opacity interact to produce screen-space alpha values. It differs from standard 3D Gaussian Splatting by offering alternative radial basis functions (Inverse Multiquadric) and decoupling the visibility threshold from the opacity parameter (Opacity-Independent Truncation) to enable more aggressive culling and consistent gradient flow.

### [Atomized Proliferation](12924-atomized-proliferation.md)
Atomized Proliferation is a densification and refinement strategy for 3D Gaussian Splatting that enhances geometric precision by constraining Gaussians representing fine details into "Atom Gaussians" (isotropic spheroids with uniform size) and prioritizing their proliferation to align with scene geometry. Unlike standard 3D Gaussian Splatting, which struggles to stabilize densification and often results in noisy meshes or blurry artifacts, Atomized Proliferation uses a warm-up strategy that aggressively densifies initially, then merges large smooth surfaces, leading to a declining and stabilizing trend in Gaussian count. It builds on [3D Gaussian Splatting](9128-3d-gaussian-splatting-3d-gaussian-splatting.md) training mechanics but modifies the cloning, splitting, and pruning logic to favor geometric alignment over arbitrary scale optimization.

### [DeSplat](12925-desplat.md)
DeSplat is a method for distractor-free novel view synthesis that decomposes the 3D Gaussian Splatting (3DGS) scene representation into two distinct sets of Gaussian primitives: static scene Gaussians ($G_s$) and per-view distractor Gaussians ($G_d$). It solves the problem of transient occluders (e.g., pedestrians, vehicles) corrupting the static 3D reconstruction by separating them from the permanent scene geometry without relying on external semantic segmentation models or pre-trained neural networks. Unlike sibling methods that use foundation models for detection or complex neural appearance encoders for separation, DeSplat relies solely on volume rendering and photometric loss, allowing independent rendering of static elements and transient components while maintaining the speed of standard Gaussian splatting.

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
360 Gaussian Splatting is a method for generating high-quality 3D spatial representations from 360-degree camera images by combining OpenSfM for sparse point cloud generation and Gaussian Splatting for dense Gaussian conversion. It solves the problem that standard Gaussian Splatting assumes perspective cameras and cannot directly process equirectangular images from 360-degree cameras, a limitation that previously forced users to crop or split images. It sets itself apart from standard implementations by modifying the differentiable rasterization module to support spherical rendering and using OpenSfM, which natively supports equirectangular inputs, unlike Colmap.

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
This family of methods addresses the excessive memory footprint and storage requirements of 3D Gaussian Splatting (3DGS) models, which typically consist of millions of Gaussian primitives with high-dimensional attributes. While standard 3DGS stores explicit per-Gaussian parameters, these optimization techniques reduce redundancy through various mechanisms: ASCII-based encoding for training I/O efficiency, sensitivity-aware vector clustering for quantization, and hybrid representations using hash grids and codebooks for inference. They differ from sibling methods like [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md) by focusing on data representation rather than rendering throughput, and from [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md) by employing structured compression rather than simple pruning.

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Voxel-based Gaussian Splatting (VoxGS) is a specialized volume representation that places isometric 3D Gaussians at the fixed centroids of a regular 3D voxel grid to enable efficient Computed Tomography (CT) volume extraction and differentiable X-ray rendering. Unlike standard 3D Gaussian Splatting (3DGS) which optimizes primitive positions and uses spherical harmonics for view-dependent color, VoxGS fixes positions and removes color coefficients, retaining only opacity, scale, and rotation to support precise density field extraction and stable optimization in feed-forward reconstruction frameworks.

## Operating characteristics

**Half-Precision Training:**
Using FP16 for feature vector memory fetches reduces register pressure in CUDA kernels, improving throughput [S1]. An automatic scaling factor is applied to loss values to maintain stability [S1].

**SH Degree Impact:**
Setting SH degree to 0 removes superfluous operations, improving training speed by ~16.57% with negligible quality loss, as higher-degree coefficients often remain unchanged during training [S5, unofficial].

**Background Color:**
Changing the background from white (1.0) to gray (0.4) reduces training time by ~21.85% on datasets like Truck, without negatively impacting PSNR or learning rate stability [S5, unofficial].

**Scene Extent Bias:**
Default scene extent calculation based on camera positions can lead to incorrect densification/pruning if cameras are clustered (e.g., orbiting a small object) [S3]. Using the corrected formula based on SfM points ensures appropriate Gaussian sizing [S3].

**Dynamic Scenes:**
Joint optimization of static Gaussians and deformation fields requires a warm-up phase (3000 iterations) for static Gaussians to stabilize before introducing deformation parameters [S4, unofficial].

## Background

3D Gaussian Splatting was introduced by Kerbl et al. in 2023 [S1, S2][S5, unofficial][S6, S7, S8, S9][S10, unofficial][S11]. The method represents scenes as a set of 3D Gaussians optimized via differentiable rasterization. Subsequent implementations and optimizations have focused on efficiency (half-precision, SH reduction), robustness (denoising, defocus handling), and extensions (mesh extraction, dynamic scenes, 360-degree support). The `gsplat` library [S2] and `tiny-cuda-nn` [S1] are common tools used in these implementations.

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
[S11] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
