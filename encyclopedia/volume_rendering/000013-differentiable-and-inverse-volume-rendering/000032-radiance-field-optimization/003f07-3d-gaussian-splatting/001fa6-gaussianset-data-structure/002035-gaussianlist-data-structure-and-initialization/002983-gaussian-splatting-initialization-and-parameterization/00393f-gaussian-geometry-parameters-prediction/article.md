# Gaussian Geometry Parameters Prediction

Also called: EVolSplat, Efficient Volume-based Gaussian Splatting, GRM, Gaussian MLP Heads, Gaussian Representation Model

## Overview
Gaussian Geometry Parameters Prediction is a feed-forward method that predicts the geometric attributes (position, opacity, covariance) of 3D Gaussians from a global volumetric feature representation, rather than regressing them per-pixel from local 2D features [S2]. It solves the problem of ghosting artifacts and multi-view inconsistencies inherent in pixel-aligned 3D Gaussian Splatting (3DGS) methods by accumulating depth information into a unified 3D volume [S2]. Unlike [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md), which relies on per-scene optimization to refine initial parameters, this method predicts a global scene representation in a single pass, improving global consistency and reducing redundant primitives in overlapping regions [S2].

## How it works
**Inputs:** A set of $N$ sparse input images $\{I_n\}_{n=1}^N$, their calibrated intrinsic matrices $\{K_n\}_{n=1}^N$, and extrinsic matrices $\{T_n = R_n | t_n\}_{n=1}^N$. An off-the-shelf metric depth estimator provides depth maps $\{D_n\}_{n=1}^N$ for each frame [S2].

**Outputs:** A set of 3D Gaussians $\mathcal{G} = \{(\mu_i, \alpha_i, \Sigma_i, c_i)\}_{i=1}^{N_p}$, where $\mu_i$ is the center, $\alpha_i$ is the opacity, $\Sigma_i$ is the covariance matrix, and $c_i$ is the view-dependent color [S2].

**Data Structures:**
1.  **Global Point Cloud ($P$):** A set of 3D points in world coordinates, $P \in \mathbb{R}^{N_p \times 3}$, constructed by unprojecting RGB images using depth maps and camera poses [S2].
2.  **Feature Volume ($F$):** A sparse 3D tensor $F \in \mathbb{R}^{H \times W \times D \times C}$ extracted by a sparse 3D CNN from the point cloud $P$, aligned with the world coordinate system [S2].
3.  **MLP Heads:** Three independent Multi-Layer Perceptrons: $\mathcal{D}_{pos}$ for position offset, $\mathcal{D}_{opa}$ for opacity, and $\mathcal{D}_{cov}$ for covariance [S2].

**Parameters:**
*   $N$: Number of input images.
*   $N_p$: Number of primitives in the global point cloud.
*   $K$: Number of nearest reference views used for retrieving 2D context (color window $\{c_k\}_{k=1}^K$) [S2].
*   $v_{\text{size}}$: Voxel size used to scale the position offset output [S2].

**Steps:**

1.  **Global Point Cloud Construction:**
    Unproject the RGB images $\{I_n\}$ into 3D space using the depth maps $\{D_n\}$ and camera poses to construct the global point cloud $P$:
    $$ P = \bigcup_{n=1}^N \pi^{-1}(I_n, D_n, T_n, K_n) $$
    where $\pi^{-1}$ denotes pixel unprojection [S2].
    Apply a uniform filter to $P$ to delete duplicate primitives while keeping the scene structure. Apply a statistical filter to remove floating artifacts (floaters) from the processed primitives [S2].

2.  **Feature Volume Extraction:**
    Feed the filtered global point cloud $P$ into a sparse 3D CNN (e.g., sparse U-Net) to extract the neural feature volume $F$ [S2].

3.  **Initialization:**
    Initialize the Gaussian centers $\mu^{init}_i$ using the points in $P$:
    $$ \mu^{init}_i = p_i \in P $$
    [S2]

4.  **Geometry Parameters Prediction (Training):**
    For each initialized Gaussian $i$:
    a. Query the latent feature $f_i$ from $F$ using trilinear interpolation at the current estimated position. During training, the position is updated iteratively. Let $\Delta \mu^{prev}_i$ be the position offset from the previous iteration (initialized to 0 for the first iteration). The query position is $\mu^{init}_i + \Delta \mu^{prev}_i$:
    $$ f_i = F(\mu^{init}_i + \Delta \mu^{prev}_i) $$
    b. Decode opacity $\alpha_i$ and covariance $\Sigma_i$ using the MLP heads:
    $$ \alpha_i = \mathcal{D}_{opa}(f_i) $$
    $$ \Sigma_i = \mathcal{D}_{cov}(f_i) $$
    c. Decode the position offset $\Delta \mu_i$:
    $$ \Delta \mu_i = \text{Tanh}(\mathcal{D}_{pos}(f_i)) \cdot v_{\text{size}} $$
    d. Update the Gaussian center:
    $$ \mu_i = \mu^{init}_i + \Delta \mu_i $$
    e. Track $\Delta \mu_i$ as $\Delta \mu^{prev}_i$ for the next iteration [S2].

5.  **Geometry Parameters Prediction (Inference):**
    a. Initialize $\Delta \mu^{prev}_i = 0$.
    b. Recursively execute the update rules (querying $F$, decoding $\alpha_i, \Sigma_i, \Delta \mu_i$) twice to obtain a converged $\Delta \mu_i$ [S2].
    c. Use the converged point $\mu_i = \mu^{init}_i + \Delta \mu_i$ to query $F$ and decode the final $\alpha_i$ and $\Sigma_i$ [S2].

6.  **Color Prediction:**
    Project the 3D Gaussians to the $K$ nearest reference views to retrieve 2D context, including a color window $\{c_k\}_{k=1}^K$ and visibility maps. Decode the Gaussian color $c_i$ using this retrieved 2D context [S2].

7.  **Background Modeling:**
    Model distant views and the sky with a generalizable hemisphere Gaussian model where geometry is fixed and color is predicted similarly to the foreground volume [S2].

8.  **Rendering:**
    Compose the foreground and background Gaussians. Render the image using alpha blending:
    $$ c = \sum_{i \in G} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $G$ is the set of 3D Gaussians, $c_i$ is the view-dependent color, and $\alpha_i$ is the opacity [S2].

**Pitfalls:**
*   **Ghosting Artifacts:** Pixel-aligned 3DGS methods generate redundant and inconsistent primitives in overlapping regions. This method addresses this by using a global volume and filtering duplicates [S2].
*   **Depth Noise:** Monocular depth predictions contain noise and floating artifacts. This method handles this by applying a statistical filter to remove floaters after uniform filtering [S2].
*   **Recursive Convergence:** The position offset $\Delta \mu_i$ depends on the previous estimation. During inference, two recursive iterations are empirically sufficient for convergence [S2].

## Subtopics

### [Textured-GS](9436-textured-gs.md)
Textured-GS enhances the textural representation of individual 3D Gaussians by modeling spatially defined color and opacity variations across the Gaussian ellipsoidal surface using Spherical Harmonics (SH), without increasing the number of Gaussians or adding extra parameters beyond the existing SH framework. Unlike standard 3D Gaussian Splatting (3DGS) where each Gaussian retains a fixed opacity and shows a single color when viewed from a specific direction, Textured-GS enables color variation across the Gaussian ellipsoidal surface and opacity variations, allowing for deviations from the standard ellipsoidal shape such as 'unidirectional Gaussians' that are opaque on one side and transparent on the other. It differs from [Gaussian Geometry Parameters Prediction](8442-gaussian-geometry-parameters-prediction.md) by using the surface intersection vector for SH evaluation instead of the viewing vector, allowing spatial variation of color/opacity within a single Gaussian.

### [Oriented Gaussian Splatting](9621-oriented-gaussian-splatting.md)
Oriented Gaussian Splatting is a variant of 3D Gaussian Splatting where each primitive is assigned a learnable oriented normal vector $n_i$ to create a reciprocal attenuation model, enabling the derivation of explicit geometric fields (normal and occupancy) via the Objects as Volumes framework. It differs from standard Gaussian Splatting by restoring reciprocity ($\sigma(x, w) = \sigma(x, -w)$), which is broken by standard attenuation, thereby allowing the extraction of watertight meshes and consistent normals.

### [3D Skew-Normal Splatting](10045-3d-skew-normal-splatting.md)
3D Skew-Normal Splatting (SNS) replaces the standard symmetric 3D Gaussian kernel with the Azzalini Skew-Normal distribution to model asymmetric structures, such as sharp boundaries and one-sided surfaces, while preserving analytical tractability under affine transformations. It sets apart from standard Gaussian Splatting and its symmetric variants (e.g., [Textured-GS](9436-textured-gs.md), [Oriented Gaussian Splatting](9621-oriented-gaussian-splatting.md)) by introducing a learnable skewness parameter that allows continuous interpolation between symmetric Gaussians and half-ellipsoids, enabling a single primitive to represent complex geometries that would otherwise require multiple overlapping kernels.

### [Deformable 3D Gaussian Splatting](10060-deformable-3d-gaussian-splatting.md)
Deformable 3D Gaussian Splatting is a method for dynamic scene reconstruction that learns 3D Gaussians in a canonical (static) space and models spatial-temporal deformation using an implicit deformation field implemented as a Multi-Layer Perceptron (MLP). It differs from static 3D Gaussian Splatting by introducing time-dependent offsets to Gaussian attributes, and from other dynamic variants like [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md) by relying on an implicit MLP rather than explicit mesh constraints or voxel grids.

### [Fourier Splatting Primitive](11094-fourier-splatting-primitive.md)
The Fourier Splatting primitive is a planar surfel for radiance field rendering that replaces the fixed Gaussian or elliptical falloff with a boundary defined by a Fourier series, allowing the representation of arbitrary closed shapes. Unlike 3D Gaussian Splatting (3DGS) or 2D Gaussian Splatting (2DGS), which are limited to fixed elliptical/disc shapes, this primitive enables higher fidelity representation of complex geometry and fine-grained details. Its key differentiator is inherent scalability: the level of detail (LoD) is controlled by truncating Fourier coefficients at runtime, allowing a single trained model to be rendered at varying complexities without retraining or post-hoc compression.

### [Conditional Slicing Mechanism](11455-conditional-slicing-mechanism.md)
The Conditional Slicing Mechanism conditions a 7-dimensional Gaussian (defined over spatial, temporal, and angular dimensions) on observed time $t$ and view direction $d$ to yield a conditional 3D Gaussian governing the spatial component. This operation "slices" the full 7D distribution to integrate temporal dynamics and view-dependent effects into the traditional 3D rendering pipeline without introducing additional temporal dependencies in the spherical harmonics. It sets itself apart from static Gaussian Splatting variants by explicitly modeling the joint distribution of space, time, and view, allowing the spatial mean and covariance to shift based on the current rendering context.

### [Structure-Aware 3D Gaussian Splatting (SAGS)](11908-structure-aware-3d-gaussian-splatting-sags.md)
Structure-Aware 3D Gaussian Splatting (SAGS) is a novel-view synthesis method that implicitly encodes scene geometry using a local-global graph representation to facilitate learning complex scenes and enforce meaningful point displacements that preserve structure, thereby reducing floating-point artifacts and storage requirements. Unlike standard 3D Gaussian Splatting which optimizes each Gaussian independently, SAGS leverages inter- and intra-connectivity between 3D point positions via Graph Neural Networks (GNNs) to predict Gaussian attributes, outperforming 3D-GS and Scaffold-GS in capturing high-frequency details while significantly reducing memory footprint.

### [Mesh-embedded representation](12559-mesh-embedded-representation.md)
Not given in the sources.

### GSGEN
GSGEN is a method introducing 3D SDS loss based on Point-E to mitigate the multi-face or Janus problem in text-to-3D generation [S1]. It adopts Point-E to initialize the point cloud as the initial geometry for optimization and refines the appearance with only the 2D image prior [S1].

### BrightDreamer
BrightDreamer is a method predicting the deviation of a set of fixed anchor points to determine the centers of Gaussians [S1]. The centers of Gaussians are determined from these predicted deviations [S1].

### NeRF-Guided 3D Gaussian Splatting
NeRF-Guided 3D Gaussian Splatting uses a pretrained NeRF as a teacher to guide feature distillation, improving alignment between geometry and semantics [S3, unofficial].

### Gaussian Splatting Attribute Prediction
Gaussian Splatting Attribute Prediction is a voxel transformer that predicts residuals added to initial Gaussian rendering attributes on voxels, rather than generating new attributes directly [S4]. The residual $\Delta \mathbf{G}_j$ is predicted by an MLP $\text{MLP}_{fine}$ on the concatenation of offset $\mathbf{O}_j$ and voxel attribute $\mathbf{E}_j$: $\Delta \mathbf{G}_j = \text{MLP}_{fine}([\mathbf{O}_j, \mathbf{E}_j])$. The final splat rendering parameter is $\mathbf{G}'_j = \mathbf{G}_j + \Delta \mathbf{G}_j$ [S4].

### 6D Gaussian Splatting
6D Gaussian Splatting is an extension of 3DGS that models view-dependent effects by adding directional dimensions, representing scene elements as 6D Gaussians (space+direction) [S5]. It parameterizes the full 6D covariance via Cholesky decomposition $\Sigma = LL^\top$ with $L$ lower-triangular and positive diagonal entries to guarantee positive definiteness [S5].

### Anisotropic Covariance
Anisotropic Covariance allows 3D Gaussians to scale anisotropically along the x, y, and z axes and rotate in 3D space, rather than remaining spherical, to better fit scene geometry [S6, unofficial]. Each Gaussian optimizes 3 scale parameters (x, y, z expansion) and 4 rotation parameters (quaternion). The Isotropic variant fixes these, removing 7 variables per Gaussian. On Average-30K, Full achieves PSNR 26.05, while the Isotropic variant achieves 25.23 [S6, unofficial]. Isotropic forces Gaussians to remain spherical, causing blurry, rounded artifacts (e.g., tree trunks/leaves appear as round dots), whereas Anisotropic allows elongated/rotated shapes matching scene geometry [S6, unofficial].

### Defocus Blur Modeling
Defocus Blur Modeling models optical defocus by adjusting the scale $s$ of 3D Gaussians based on their distance from the camera's focal plane [S7]. During training, larger scale values $\bar{s}_j$ are used to rasterize blurred images. During testing, smaller scale values $s_j$ are used to render sharp details [S7]. Unlike a naive Gaussian kernel that blurs the entire image uniformly, this method selectively enlarges covariances only for Gaussians corresponding to blurred regions [S7].

## Operating characteristics
*   **Time Complexity:** Feed-forward inference is fast as it avoids per-scene optimization. The method involves a sparse 3D CNN pass and MLP decoding, which is significantly faster than iterative optimization-based 3DGS [S2].
*   **Memory Complexity:** By downsampling initial primitives and using a global volume, it reduces computation and memory compared to per-pixel aligned methods [S2].
*   **Performance:** EVolSplat achieves real-time rendering by feed-forward prediction of Gaussian primitives [S2]. It improves global consistency and reduces ghosting artifacts compared to existing feed-forward methods like MVSplat, Flash3D, and DrivingForward [S2].
*   **Strengths:** Handles foreground and background separately; robust to noisy depth predictions via filtering; reduces redundancy in overlapping regions [S2].
*   **Limitations:** Relies on the quality of the off-the-shelf depth estimator; may lose high-frequency appearance details due to downsampling, mitigated by IBR-based color prediction [S2].
*   **When to use:** Efficient urban scene reconstruction from sparse views where real-time rendering and global consistency are required [S2].
*   **When not to use:** Scenes requiring per-pixel local detail that might be lost in global volume downsampling, or where high-quality metric depth is unavailable [S2].

## Background
Gaussian Geometry Parameters Prediction, specifically in the form of EVolSplat, was introduced by Sheng Miao, Jiaxin Huang, Dongfeng Bai, Xu Yan, Hongyu Zhou, Yue Wang, Bingbing Liu, Andreas Geiger, and Yiyi Liao from Zhejiang University, Huawei Noah’s Ark Lab, and the University of Tübingen [S2]. It was presented at CVPR 2025 [S2]. The method addresses the limitations of existing NeRF and 3DGS-based methods that require slow, per-scene optimization and struggle with large camera movements and small parallax in driving scenes [S2]. It builds on the idea of accumulating depth information in 3D space to predict a global scene representation, contrasting with pixel-aligned methods that use local cost volumes or epipolar transformers [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S4] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
[S5] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
[S6] GaussianSplatting論文のAblationについての解説・考察, https://qiita.com/kotauchisunsun/items/f1e89f28aa770b8d3b19, fetched 2026-10-29 (unofficial)
[S7] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
