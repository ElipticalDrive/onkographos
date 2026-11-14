# 3D Gaussian Splatting Implementation Details

Also called: 3DGS CUDA kernel implementation, GS Implementation Parameters, Gaussian Splatting Training Configuration, Implementation Details, gsplat implementation details, half-precision rasterization implementation

## Overview
This article details the specific implementation parameters, optimization strategies, and variant algorithms required to train and render 3D Gaussian Splatting (3DGS) scenes effectively. It addresses the gap between the theoretical differentiable rasterization framework and practical execution, covering hardware-aware optimizations (half-precision kernels), scene initialization corrections, and specialized handling for defocus, noise, underwater scattering, and large-scale constraints. It builds on [Optimization and Parallelization](6233-optimization-and-parallelization.md) by providing the concrete hyperparameters and algorithmic modifications used in state-of-the-art implementations.

## How it works

### Standard Training Configuration
The baseline training process utilizes the `gsplat` library [S2] with the 3DGUT formulation [S1]. The optimizer is Adam with a learning rate of $1 \times 10^{-3}$ [S2]. Spherical Harmonics (SH) degree is typically set to 1 for simplicity and efficiency [S2]. During each iteration, a single image is randomly selected from the training set for supervision [S2].

To ensure stability in half-precision (FP16) training, an automatic scaling factor is applied to the loss values [S1]. Feature vectors in the rasterization kernels (both forward and backward passes) are fetched using half-precision memory accesses to reduce register pressure [S1]. Neural network components, such as appearance MLPs, are implemented using tiny-cuda-nn’s JIT-compiled cooperative vector MLPs, trained and evaluated in FP16 [S1].

### Scene Extent Correction
The default 3DGS scene extent $e_{\text{scene}}$ is calculated based on camera positions, which can bias densification if cameras are clustered (e.g., orbiting a small object) [S3]. The corrected scene extent $e'_{\text{scene}}$ uses the Structure-from-Motion (SfM) point cloud:

$$
\bar{C} = \frac{1}{N_{\text{cam}}} \sum_{i=1}^{N_{\text{cam}}} C_i
$$

$$
e'_{\text{scene}} = \frac{1}{N_{\text{SfM}}} \sum_{i=1}^{N_{\text{SfM}}} \|\bar{C} - p_i\|_2
$$

where $N_{\text{cam}}$ is the number of cameras, $C_i$ is the $i$-th camera position, $N_{\text{SfM}}$ is the number of SfM points, and $p_i$ is the $i$-th SfM point [S3].

### Densification and Multi-Scale Constraints
For large-scale scenes, a position-aware multi-scale constraint adjusts densification thresholds based on the viewer's distance [S8]. Let $\mu$ be the Gaussian center, $c$ be the camera center, and $r$ be the scene radius. The scaling factor $\gamma(\mu)$ is:

$$
\gamma(\mu) = 
\begin{cases} 
1 & \text{if } \|\mu - c\|^2 < 2r \\
\frac{\|\mu - c\|^2}{r} - 1 & \text{if } \|\mu - c\|^2 \ge 2r
\end{cases}
$$

Densification operations are triggered as follows [S8]:
1.  **Clone:** If $\max(s) < g \cdot \gamma(\mu)$, where $g$ is the cloning threshold.
2.  **Split:** If $\max(s) \ge g \cdot \gamma(\mu)$.
3.  **Prune:** If $\max(s) > b \cdot \gamma(\mu)$, where $b$ is the elimination threshold.

### Selective Defocus Blurring
To handle spatially varying blur, a compact MLP predicts a per-Gaussian scale offset $\delta s_j$ [S4]. The covariance is transformed using this offset, and the resulting 2D projection acts as a pixel-wise blur kernel during rasterization. If $\delta s_j$ is constant for all Gaussians, the method degrades to a uniform Gaussian blur [S4].

### View-Dependent Opacity Kernel
To model specular reflectance, a view-dependent opacity kernel adds 5 learnable parameters ($\alpha, \theta, T, \beta$) to the standard Gaussian [S6, unofficial]. The opacity $\hat{\alpha}$ is:

$$
\hat{\alpha}(\theta, \beta, T, \alpha) = \alpha \cdot \frac{\cos\left(\max\left(0, \min\left(\frac{2\theta}{T}, \pi\right)\right)\right) + 1}{2} \exp(\beta)
$$

where $\theta$ is the angle between the view direction and the lobe's central orientation, $T$ controls the angular span, and $\beta$ modulates sharpness [S6, unofficial]. As $\theta$ increases, transparency diminishes to zero, preventing artifacts outside the lobe's influence [S6, unofficial].

### Underwater Rendering and Backscatter Estimation
For underwater scenes, the rendering equation incorporates direct attenuation $B_d$ and backscatter parameters $B_\infty, B_b$ [S7]. The camera sensor must have a linear response; gamma-corrected images must be linearized [S7].

**Backscatter Estimation Algorithm:**
1.  Divide the current depth map into 10 evenly-spaced clusters from $\min(z)$ to $\max(z)$ [S7].
2.  In each cluster, identify pixels in the ground-truth image with RGB triplets in the bottom 1st percentile [S7].
3.  These triplets serve as an overestimate of backscatter $B_s = B_\infty (1 - e^{-B_b z})$ [S7].
4.  Use nonlinear least squares fitting on pairs $(B_s, z)$ to estimate $B_\infty \in [0, 1]$ and $B_b \in [0, 5]$ [S7].
5.  Re-estimate these parameters every 500 optimization steps [S7].

Depth $z$ is extracted from the 3DGS scene at each pixel during rendering [S7]:

$$
z = \frac{\sum_{i=1}^M d_i \alpha_i T_i}{\sum_{i=1}^M \alpha_i T_i}
$$

where $d_i$ is the depth of Gaussian $i$ [S7].

### Denoise-GS Training Procedure
For sparse-view and noisy scenes, a two-round self-supervised denoising process is used [S5]:
1.  **Round 1:** Train a UNet using Noise2Void (N2V) for denoising. Process denoised images through InstantSplat to initialize and train 3DGS. Render and filter results by viewpoint to ensure representativeness [S5].
2.  **Round 2:** Fine-tune the UNet using a composite loss balancing low-frequency and perceptual detail consistency. Render again on selected viewpoints to generate denoised views [S5].

### Volume Setup for Urban Scenes
For urban environments (e.g., Waymo), the foreground volume is defined by an axis-aligned bounding box. Typical dimensions include height 12.8m, width 32m, and length 64m, with a voxel size of 0.1m, resulting in a volume dimension of $128 \times 320 \times 640$ [S2]. For zero-shot inference on Waymo, the range may be adjusted to $(50\text{m}, 20\text{m}, 128\text{m})$ [S2].

## Subtopics

### [Continuous Attenuation Model for 3D Gaussian Splatting](9624-continuous-attenuation-model-for-3d-gaussian-splatting.md)
Reformulates rasterization into a volumetric ray-marching framework using a continuous attenuation field $\sigma(\mathbf{x}, \mathbf{w})$. Ensures unbiased, ellipsoidal transmittance isosurfaces, preventing surface erosion and multi-view inconsistency found in prior approximations like GGGS [9624-continuous-attenuation-model-for-3d-gaussian-splatting.md].

### DAVIGS
A plug-and-play appearance modeling method that mitigates floaters and color distortions by transforming rendered images using 3D consistent features queried from multi-resolution hash grids. Decouples global view-dependent variations from local spatial changes, maintaining real-time speeds by discarding the appearance module after optimization [12021-davigs.md].

### [Sorting and Rendering Pipeline Optimization](12180-sorting-and-rendering-pipeline-optimization.md)
Addresses computational bottlenecks in 3DGS rendering by mapping software strategies to hardware constraints (e.g., Roofline model) or replacing standard sorting with hardware-friendly, comparison-free logic and sorting-free approximations [12180-sorting-and-rendering-pipeline-optimization.md].

### [LangSplat Training Procedure](12438-langsplat-training-procedure.md)
Optimizes 3DGS scenes to incorporate semantic language features (e.g., CLIP) alongside photometric rendering. Distinguished by conditional inclusion of language loss and disabling of densification/pruning during language-aware training [12438-langsplat-training-procedure.md].

### [Content-Aware Texturing for Gaussian Splatting](12606-content-aware-texturing-for-gaussian-splatting.md)
Augments 2D Gaussians with spatially varying texture maps that offset SH colors. Uses dynamic, content-aware texel allocation and splitting-based densification to avoid convergence issues associated with cloning textured primitives [12606-content-aware-texturing-for-gaussian-splatting.md].

### [Directed Distance Field (DDF) for Gaussian Splatting](12854-directed-distance-field-ddf-for-gaussian-splatting.md)
A neural field mapping rays to distance and visibility for constant-time, constant-memory ray queries (shadows, AO, GI). Eliminates the need for mesh extraction or BVH traversal, scaling independently of scene size compared to BVH-based tracing [12854-directed-distance-field-ddf-for-gaussian-splatting.md].

### [Gaussian Representation and Truncation](12855-gaussian-representation-and-truncation.md)
Modifies spatial extent, density, and survival criteria of Gaussians using algebraic decay functions, opacity-independent culling, and geometry-guided proliferation to improve geometric fidelity and reduce redundancy [12855-gaussian-representation-and-truncation.md].

### [DeSplat](12925-desplat.md)
Decomposes 3DGS into static ($G_s$) and per-view distractor ($G_d$) Gaussians to remove transient occluders (pedestrians, vehicles) without external semantic models. Relies solely on volume rendering and photometric loss [12925-desplat.md].

### [360 Gaussian Splatting](12929-360-gaussian-splatting.md)
Generates 3D representations from 360-degree equirectangular images by modifying the differentiable rasterizer for spherical rendering and using OpenSfM, which natively supports equirectangular inputs unlike Colmap [12929-360-gaussian-splatting.md].

### [Data Compression and Storage Optimization](12943-data-compression-and-storage-optimization.md)
Reduces the >1 GB memory footprint of 3DGS by compressing numerical representations, clustering parameters into codebooks, or replacing explicit attributes with neural fields. Focuses on data density rather than rendering throughput [12943-data-compression-and-storage-optimization.md].

### [Voxel-based Gaussian Splatting](13071-voxel-based-gaussian-splatting.md)
Places isometric 3D Gaussians at fixed voxel grid centroids to enable efficient CT volume extraction and differentiable X-ray rendering. Fixes positions and removes color coefficients, retaining only opacity, scale, and rotation [13071-voxel-based-gaussian-splatting.md].

### [Bézier-boundary gradient approximation](13137-b-zier-boundary-gradient-approximation.md)
Approximates gradients of discontinuous indicator functions with respect to Bézier curve control points, enabling backpropagation in discontinuity-aware rendering where standard differentiable rendering fails [13137-b-zier-boundary-gradient-approximation.md].

### [Discontinuity-aware Gaussian Splatting](13138-discontinuity-aware-gaussian-splatting.md)
Enables 3DGS to render sharp boundaries by introducing "pre-scissoring" along cubic Bézier curves and using Bézier-boundary gradient approximation to maintain differentiability [13138-discontinuity-aware-gaussian-splatting.md].

### [SOGS (Second-Order Anchor for Advanced 3D Gaussian Splatting)](13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md)
Uses covariance-based feature augmentation to capture co-variation patterns, achieving superior quality with smaller feature vectors compared to standard anchor methods like Scaffold-GS [13171-sogs-second-order-anchor-for-advanced-3d-gaussian-splatting.md].

### [Gaussian Voxel Kernel Functions (GVKF)](13979-gaussian-voxel-kernel-functions-gvkf.md)
Replaces discrete opacity values with Gaussian kernel functions for continuous scene representation. Enables direct mesh extraction via bidirectional mapping between opacity field and surface, reducing storage and training memory [13979-gaussian-voxel-kernel-functions-gvkf.md].

### [Underwater 3D Gaussian Splatting with Backscatter Loss](13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md)
Adapts 3DGS to underwater environments by learning direct attenuation and backscatter coefficients. Uses alternating optimization to refine depth and medium parameters simultaneously, improving quality at large distances [13980-underwater-3d-gaussian-splatting-with-backscatter-loss.md].

### [Error-Driven 2D Gaussian Training](13985-error-driven-2d-gaussian-training.md)
Densifies 2D Gaussian kernels in image space using a differentiable rasterizer, then back-projects to 3D for initialization. Solves view-dependent kernel initialization for complex specular radiance where standard 3D initialization is insufficient [13985-error-driven-2d-gaussian-training.md].

### [Mesh Extraction and Representation](14095-mesh-extraction-and-representation.md)
[14095-mesh-extraction-and-representation.md]

### [Spherical Harmonics Optimization](14206-spherical-harmonics-optimization.md)
Reduces computational cost and memory of view-dependent color representation by compressing, quantizing, or distilling SH coefficients. Targets the algebraic structure of SH distinct from general rendering optimizations [14206-spherical-harmonics-optimization.md].

### [TraGraph-GS](14454-tragraph-gs.md)
Uses graph-based adaptive spatial partitioning and regularization for large-scale scenes. Decouples scenes into local regions and a coarse global representation, using camera connectivity graphs to ensure balanced subgraphs and eliminate boundary artifacts [14454-tragraph-gs.md].

### [Explicit 4D Gaussian Splatting](14584-explicit-4d-gaussian-splatting.md)
Achieves memory efficiency in dynamic scenes by separating Gaussians into static and dynamic categories. Explicitly samples positions and rotations at sparse timestamps and uses interpolation for continuous motion, avoiding memory overhead of multiple primitives [14584-explicit-4d-gaussian-splatting.md].

### [3D Gaussian Particle Approximation of VDB Datasets](14723-3d-gaussian-particle-approximation-of-vdb-datasets.md)
Converts OpenVDB volumetric datasets into non-overlapping 3D Gaussian particles for memory-efficient, lossy representation with LOD support. Enforces non-overlapping Gaussians derived from OpenVDB leaf nodes for deterministic spatial partitioning [14723-3d-gaussian-particle-approximation-of-vdb-datasets.md].

### [Color and Appearance Modeling](14740-color-and-appearance-modeling.md)
Extends radiance representation to handle complex lighting and view-dependent effects by introducing external conditioning mechanisms (neural fields, illumination grids). Disentangles geometry from appearance to solve blurry specular highlights and transient artifacts [14740-color-and-appearance-modeling.md].

## Operating characteristics

**Time and Memory Complexity:**
Standard 3DGS training is memory-intensive due to per-Gaussian attributes (position, covariance, opacity, SH coefficients), often exceeding 1 GB for large scenes [12943-data-compression-and-storage-optimization.md]. Half-precision rasterization reduces register pressure and memory bandwidth usage [S1]. The `gsplat` library provides efficient CUDA kernels for forward and backward passes [S2].

**Measured Performance:**
In urban view synthesis (KITTI-360, Waymo), methods like EVolSplat achieve PSNR of 23.26 dB and 23.43 dB respectively, with FPS of 83.81 and memory usage of 10.41 GB [S2].

**Strengths and Limitations:**
Strengths include real-time rendering speeds, high fidelity, and efficient CUDA implementation [S1, S2]. Limitations include sensitivity to camera distribution for scene extent [S3], difficulty with specular highlights using standard SH [14740-color-and-appearance-modeling.md], and artifacts in sparse-view or noisy scenarios [S5].

**When to Use:**
Use for high-quality, real-time novel view synthesis from dense image collections. Use variants for specific challenges: underwater (Backscatter), defocus (Selective Defocus), sparse views (Denoise-GS), or large-scale scenes (TraGraph-GS, Multi-Scale Constraint).

**When Not to Use:**
Avoid for scenes with significant transient occluders unless using DeSplat. Avoid for equirectangular inputs unless using 360 Gaussian Splatting. Avoid if linear camera response is not available for underwater/backscatter methods [S7].

## Background

3D Gaussian Splatting was introduced by Kerbl et al. [26] as a real-time explicit scene representation. The implementation details described here stem from subsequent optimizations and adaptations for specific domains (underwater, large-scale, sparse-view) and hardware constraints (half-precision, CUDA kernels). The `gsplat` library [S2] and 3DGUT formulation [S1] represent the current standard for efficient implementation.

## Sources

[S1] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
[S4] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S5] Denoise-GS: Self-Supervised Denoising for Sparse-View 3D Gaussian Splatting., https://doi.org/10.3390/s26020651, fetched 2026-10-26
[S6] Augmented Radiance Field: A General Framework for Enhanced Gaussian Splatting, https://doi.org/10.48550/arXiv.2602.19916, fetched 2026-10-30 (unofficial)
[S7] Gaussian splashing enables direct volumetric rendering underwater, https://doi.org/10.1038/s41598-026-56917-4, fetched 2026-10-27
[S8] TraGraph-GS: Trajectory Graph-based Gaussian Splatting for Arbitrary Large-Scale Scene Rendering., https://doi.org/10.1109/tpami.2026.3709368, fetched 2026-10-26
