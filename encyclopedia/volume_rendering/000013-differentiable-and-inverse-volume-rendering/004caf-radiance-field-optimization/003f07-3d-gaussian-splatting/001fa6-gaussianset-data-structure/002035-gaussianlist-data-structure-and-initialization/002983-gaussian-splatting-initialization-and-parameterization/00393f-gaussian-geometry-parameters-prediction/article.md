# Gaussian Geometry Parameters Prediction

Also called: EVolSplat, Efficient Volume-based Gaussian Splatting, GRM, Gaussian MLP Heads, Gaussian Representation Model

## Overview
Gaussian Geometry Parameters Prediction is a feed-forward method that predicts the geometric attributes (position, opacity, covariance) of 3D Gaussians directly from a global volumetric feature representation, rather than optimizing them iteratively per scene [S2]. It solves the problem of multi-view inconsistencies and ghosting artifacts inherent in pixel-aligned prediction methods by accumulating depth information into a unified 3D volume [S2]. Unlike [Gaussian Splatting Initialization and Parameterization](2313-gaussian-splatting-initialization-and-parameterization.md), which sets initial parameters for optimization, this method produces the final parameters in a single pass using a 3D convolutional network and MLP heads [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of $N$ sparse input images $\{I_n\}_{n=1}^N$, their calibrated intrinsic matrices $\{K_n\}$, extrinsic matrices $\{T_n = R_n | t_n\}$, and monocular depth estimates $\{D_n\}$ [S2].
*   **Output:** A set of 3D Gaussians $\mathcal{G} = \{(\mu_i, \alpha_i, \Sigma_i, c_i)\}_{i=1}^{N_p}$, where $\mu_i$ is the center, $\alpha_i$ is opacity, $\Sigma_i$ is the covariance matrix, and $c_i$ is the view-dependent color [S2].

**Data Structures:**
*   **Global Point Cloud ($P$):** A set of 3D points in world coordinates, $P \in \mathbb{R}^{N_p \times 3}$, constructed by unprojecting RGB-D data [S2].
*   **Feature Volume ($F$):** A sparse 3D tensor $F \in \mathbb{R}^{H \times W \times D \times C}$ extracted by a sparse 3D CNN, aligned with the world coordinate system [S2].
*   **MLP Heads:** Three independent Multi-Layer Perceptrons: $\mathcal{D}_{pos}$ (position offset), $\mathcal{D}_{opa}$ (opacity), and $\mathcal{D}_{cov}$ (covariance) [S2].

**Steps:**
1.  **Depth Estimation and Filtering:** Use an off-the-shelf metric depth estimator to predict depth maps $\{D_n\}$ for each input image $\{I_n\}$ [S2]. Apply a depth consistency check and 3D filters to remove floating artifacts [S2].
2.  **Global Point Cloud Construction:** Unproject the RGB images and depth maps into 3D space using the camera matrices to form the initial point cloud $P$:
    $$ P = \bigcup_{n=1}^N \pi^{-1}(I_n, D_n, T_n, K_n) $$
    where $\pi^{-1}$ denotes pixel unprojection [S2].
3.  **Denoising:** Apply a uniform filter to $P$ to delete duplicate primitives while preserving scene structure, followed by a statistical filter to remove remaining floaters [S2].
4.  **Feature Volume Extraction:** Feed the filtered point cloud $P$ into a sparse 3D CNN (e.g., sparse U-Net) to extract the neural feature volume $F$ [S2].
5.  **Geometry Parameter Prediction:**
    *   Initialize Gaussian centers $\mu^{init}_i$ using the points in $P$ (i.e., $\mu^{init}_i = p_i \in P$) [S2].
    *   For each Gaussian $i$, query the feature volume $F$ at the current estimated position to obtain a feature vector $f_i$. During training, this position is $\mu^{init}_i + \Delta \mu^{prev}_i$, where $\Delta \mu^{prev}_i$ is the offset from the previous iteration [S2].
    *   Decode the geometry attributes using the MLP heads:
        $$ \alpha_i = \mathcal{D}_{opa}(f_i) $$
        $$ \Sigma_i = \mathcal{D}_{cov}(f_i) $$
        $$ \Delta \mu_i = \text{Tanh}(\mathcal{D}_{pos}(f_i)) \cdot v_{\text{size}} $$
        $$ \mu_i = \mu^{init}_i + \Delta \mu_i $$
        where $v_{\text{size}}$ is the voxel size used to scale the position offset [S2].
    *   Track $\Delta \mu_i$ from the last iteration as $\Delta \mu^{prev}_i$ for the next training step [S2].
6.  **Inference Convergence:** During inference, initialize $\Delta \mu^{prev}_i = 0$. Recursively execute the position update and feature query steps (Eq. 4 and 6 in [S2]) twice to obtain a converged $\Delta \mu_i$. Use this converged position to query $f_i$ and decode $\alpha_i$ and $\Sigma_i$ [S2].
7.  **Color Prediction:** Project the 3D Gaussians to the nearest $K$ reference views to retrieve 2D context (color windows $\{c_k\}$ and visibility maps). Decode the Gaussian color $c_i$ using this 2D context [S2].
8.  **Background Modeling:** Model distant views and the sky with a generalizable hemisphere Gaussian model where geometry is fixed and color is predicted similarly to the foreground [S2].
9.  **Rendering:** Compose the foreground and background Gaussians to reconstruct the image using alpha blending:
    $$ c = \sum_{i \in \mathcal{G}} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    where $c$ is the rendered color, and the product term represents transmittance [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def predict_gaussian_geometry(input_images, depth_maps, intrinsics, extrinsics, feature_volume, voxel_size):
    """
    Predicts 3D Gaussian parameters from a global feature volume.
    
    Args:
        input_images: List of N RGB images.
        depth_maps: List of N depth maps.
        intrinsics: List of intrinsic matrices K_n.
        extrinsics: List of extrinsic matrices T_n.
        feature_volume: Sparse 3D tensor F.
        voxel_size: Scaling factor for position offset.
        
    Returns:
        Gaussians: List of Gaussian parameters (mu, alpha, Sigma, color).
    """
    # Step 1 & 2: Construct Global Point Cloud P
    P = []
    for n in range(len(input_images)):
        # Unproject RGB and Depth to 3D world coordinates
        points_3d = unproject_3d(input_images[n], depth_maps[n], intrinsics[n], extrinsics[n])
        P.extend(points_3d)
    
    # Step 3: Denoising
    P = uniform_filter(P)
    P = statistical_filter(P)
    
    # Step 5: Initialize Gaussians from P
    gaussians = []
    for p in P:
        mu_init = p
        # Initialize previous offset to zero for first iteration
        delta_mu_prev = np.zeros(3)
        
        # Inference: Recursive convergence (2 iterations sufficient)
        for iteration in range(2):
            # Query feature volume at current estimated position
            current_pos = mu_init + delta_mu_prev
            f_i = trilinear_interpolate(feature_volume, current_pos)
            
            # Decode geometry attributes
            alpha_i = decode_opacity(f_i)
            Sigma_i = decode_covariance(f_i)
            
            # Predict position offset
            raw_offset = decode_position_offset(f_i)
            delta_mu_i = tanh(raw_offset) * voxel_size
            
            # Update position
            mu_i = mu_init + delta_mu_i
            
            # Update previous offset for next iteration (or use converged value)
            delta_mu_prev = delta_mu_i
            
        # Step 7: Color Prediction (simplified)
        # Project to nearest views to get 2D context
        color_i = decode_color_from_2d_context(mu_i, Sigma_i, input_images)
        
        gaussians.append({
            'mu': mu_i,
            'alpha': alpha_i,
            'Sigma': Sigma_i,
            'color': color_i
        })
        
    return gaussians
```

**Parameters:**
*   $N$: Number of input images [S2].
*   $K$: Number of nearest reference views for color retrieval [S2].
*   $v_{\text{size}}$: Voxel size, scaling the Tanh output of the position offset [S2].

**Edge Cases and Pitfalls:**
*   **Ghosting Artifacts:** Pixel-aligned methods generate redundant primitives in overlapping regions. This method addresses this by using a global volume and filtering duplicates [S2].
*   **Depth Noise:** Monocular depth predictions contain noise and floating artifacts. Statistical filtering is required to remove floaters before volume construction [S2].

## Subtopics

### GSGEN
A method introducing 3D SDS loss based on Point-E to mitigate the multi-face or Janus problem in text-to-3D generation [S1]. It adopts Point-E to initialize the point cloud as the initial geometry for optimization and refines appearance with only the 2D image prior [S1].

### BrightDreamer
A method predicting the deviation of a set of fixed anchor points to determine the centers of Gaussians [S1]. The centers are determined from these predicted deviations [S1].

### NeRF-Guided 3D Gaussian Splatting
Uses a pretrained NeRF as a teacher to guide feature distillation, improving alignment between geometry and semantics [S3, unofficial].

### Gaussian Splatting Attribute Prediction
A voxel transformer that predicts residuals added to initial Gaussian rendering attributes on voxels, rather than generating new attributes directly [S4]. The residual is calculated as $\Delta \mathbf{G}_j = \text{MLP}_{fine}([\mathbf{O}_j, \mathbf{E}_j])$, where $\mathbf{O}_j$ is the offset and $\mathbf{E}_j$ is the voxel attribute. The final parameter is $\mathbf{G}'_j = \mathbf{G}_j + \Delta \mathbf{G}_j$ [S4].

### 6D Gaussian Splatting
An extension of 3DGS that models view-dependent effects by adding directional dimensions, representing scene elements as 6D Gaussians (space+direction) [S5]. It parameterizes the full 6D covariance via Cholesky decomposition $\Sigma = LL^\top$ with $L$ lower-triangular and positive diagonal entries to guarantee positive definiteness [S5].

### Anisotropic Covariance
Allows 3D Gaussians to scale anisotropically along the x, y, and z axes and rotate in 3D space, rather than remaining spherical, to better fit scene geometry [S6, unofficial]. Each Gaussian optimizes 3 scale parameters and 4 rotation parameters (quaternion). The Isotropic variant fixes these, removing 7 variables per Gaussian. On Average-30K, Full achieves PSNR 26.05, while Isotropic achieves 25.23 [S6, unofficial]. Isotropic forces Gaussians to remain spherical, causing blurry artifacts (e.g., tree trunks appear as round dots), whereas Anisotropic allows elongated shapes matching scene geometry [S6, unofficial].

### Defocus Blur Modeling
Models optical defocus by adjusting the scale $s$ of 3D Gaussians based on their distance from the camera's focal plane [S7]. During training, larger scale values $\bar{s}_j$ are used to rasterize blurred images; during testing, smaller scale values $s_j$ are used to render sharp details [S7]. Unlike a naive Gaussian kernel that blurs the entire image uniformly, this method selectively enlarges covariances only for Gaussians corresponding to blurred regions [S7].

### 3D Gaussians Prediction from Voxels
Predicts a set of learnable Gaussian parameters for each voxel $v$, including center offset, opacity, covariance, and spherical harmonic color [S8]. The final center is $\mu_j = r \cdot (\sigma(\bar{\mu}_j) - 0.5) + \text{Center}_j$, where $\sigma$ is sigmoid, $\bar{\mu}_j$ is the predicted offset, $\text{Center}_j$ is the voxel centroid, and $r$ is a scaling factor (set to 3 times the voxel size) [S8]. The output dimension per Gaussian is 38 (offset 3, opacity 1, covariance 6, SH color 28) [S8]. The sigmoid activation restricts the learnable offset to a localized neighborhood; the -0.5 shift facilitates symmetrical, bi-directional movement from the voxel center [S8].

### GS-LRM / Long-LRM
Predicts 3D Gaussians directly (not triplane -> Gaussians), resulting in faster downstream rendering [S9, unofficial].

## Operating characteristics
*   **Time Complexity:** Feed-forward inference is significantly faster than iterative optimization methods, as it avoids per-scene optimization loops [S2].
*   **Memory Complexity:** Depends on the resolution of the feature volume $F$ and the number of initial primitives $N_p$ after filtering [S2].
*   **Performance:** Achieves real-time rendering by feed-forward prediction of Gaussian primitives [S2].
*   **Strengths:** Improves global consistency and reduces ghosting artifacts compared to pixel-aligned methods by accumulating depth in 3D space [S2]. Handles foreground and background separately with a hemisphere model for distant views [S2].
*   **Limitations:** Relies on the quality of monocular depth estimators; noisy depth can lead to inaccurate initial point clouds, requiring robust filtering [S2].
*   **When to use:** Urban scene reconstruction and novel view synthesis from sparse views where speed and global consistency are prioritized over per-scene fine-tuning [S2].
*   **When not to use:** Scenes with very dense camera overlaps where pixel-aligned methods might suffice, or when iterative optimization is acceptable for higher fidelity [S2].

## Background
*   **Origin:** EVolSplat was introduced by Sheng Miao, Jiaxin Huang, et al. in 2025 [S2].
*   **History:** It addresses limitations of existing feed-forward 3DGS methods like MVSplat, Flash3D, and DrivingForward, which predict per-pixel Gaussians using local cost volumes or epipolar transformers, often leading to inconsistencies when accumulated globally [S2].
*   **Usage:** Applied to urban view synthesis for autonomous driving applications [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S4] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
[S5] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
[S6] GaussianSplatting論文のAblationについての解説・考察, https://qiita.com/kotauchisunsun/items/f1e89f28aa770b8d3b19, fetched 2026-10-29 (unofficial)
[S7] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
[S8] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
[S9] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
