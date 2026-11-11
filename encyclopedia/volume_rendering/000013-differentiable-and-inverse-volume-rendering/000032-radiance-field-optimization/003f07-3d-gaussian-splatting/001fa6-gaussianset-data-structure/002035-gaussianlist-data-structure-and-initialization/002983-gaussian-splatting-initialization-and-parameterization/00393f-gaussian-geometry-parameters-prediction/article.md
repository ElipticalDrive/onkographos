# Gaussian Geometry Parameters Prediction

Also called: EVolSplat, Efficient Volume-based Gaussian Splatting, GRM, Gaussian MLP Heads, Gaussian Representation Model

## Overview
Gaussian Geometry Parameters Prediction is a feed-forward method that predicts the geometric attributes (position, opacity, covariance) of 3D Gaussians from a global volumetric feature representation, rather than regressing them per-pixel from 2D feature maps [S2]. It solves the problem of ghosting artifacts and multi-view inconsistencies inherent in pixel-aligned Gaussian prediction methods by accumulating depth information into a unified 3D volume and refining Gaussian centers via iterative offset prediction [S2]. Unlike standard initialization which sets fixed parameters, this method learns to adjust initial noisy positions to accurate surface locations using a recursive query mechanism [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ sparse input images $\{I_n\}_{n=1}^N$, their calibrated intrinsic matrices $\{K_n\}$, extrinsic matrices $\{T_n = R_n | t_n\}$, and optionally monocular depth estimates $\{D_n\}$ [S2].
*   **Outputs:** A set of 3D Gaussians $\mathcal{G} = \{(\mu_i, \alpha_i, \Sigma_i, c_i)\}_{i=1}^{N_p}$, where $\mu_i$ is the center, $\alpha_i$ is opacity, $\Sigma_i$ is the covariance matrix, and $c_i$ is the view-dependent color [S2].

**Data Structures:**
1.  **Global Point Cloud ($P$):** A set of 3D points in world coordinates, $P \in \mathbb{R}^{N_p \times 3}$, constructed by unprojecting RGB-D data [S2].
2.  **Feature Volume ($F$):** A sparse 3D tensor $F \in \mathbb{R}^{H \times W \times D \times C}$ extracted by a sparse 3D CNN, aligned with the world coordinate system [S2].
3.  **MLP Heads:** Three independent Multi-Layer Perceptrons:
    *   $\mathcal{D}_{pos}$: Predicts position offset $\Delta \mu_i$.
    *   $\mathcal{D}_{opa}$: Predicts opacity $\alpha_i$.
    *   $\mathcal{D}_{cov}$: Predicts covariance matrix $\Sigma_i$ [S2].

**Parameters:**
*   $N$: Number of input images [S2].
*   $N_p$: Number of primitives in the global point cloud [S2].
*   $K$: Number of nearest reference views used for color retrieval [S2].
*   $v_{\text{size}}$: Voxel size used to scale the position offset output [S2].

**Algorithm Steps:**

1.  **Depth Estimation and Filtering:**
    *   Predict metric depth maps $\{D_n\}$ for each input image $I_n$ using an off-the-shelf estimator (e.g., Metric3D) [S2].
    *   Apply a depth consistency check to remove inconsistent values [S2].
    *   Unproject the RGB images and depth maps into 3D space using $K_n$ and $T_n$ to form a raw point cloud:
        $$ P_{\text{raw}} = \bigcup_{n=1}^N \pi^{-1}(I_n, D_n, T_n, K_n) $$
        where $\pi^{-1}$ denotes pixel unprojection [S2].
    *   Apply a uniform filter to $P_{\text{raw}}$ to delete duplicate primitives while preserving scene structure [S2].
    *   Apply a statistical filter to remove floating artifacts (floaters) from the filtered points, resulting in the global point cloud $P$ [S2].

2.  **Feature Volume Construction:**
    *   Construct a sparse tensor from $P$ [S2].
    *   Pass the sparse tensor through a sparse 3D CNN (e.g., sparse U-Net) to extract the feature volume $F$ [S2].

3.  **Initialization:**
    *   Initialize Gaussian centers using the global point cloud: $\mu^{init}_i = p_i \in P$ for each $i \in \{1, \dots, N_p\}$ [S2].
    *   Initialize the previous position offset $\Delta \mu^{prev}_i = 0$ [S2].

4.  **Geometry Parameters Prediction (Training and Inference):**
    *   For each Gaussian $i$, query the feature volume $F$ at the current estimated position $\mu^{init}_i + \Delta \mu^{prev}_i$ using trilinear interpolation to obtain the feature vector $\mathbf{f}_i$:
        $$ \mathbf{f}_i = F(\mu^{init}_i + \Delta \mu^{prev}_i) $$
    *   Predict opacity and covariance using the MLP heads:
        $$ \alpha_i = \mathcal{D}_{opa}(\mathbf{f}_i), \quad \Sigma_i = \mathcal{D}_{cov}(\mathbf{f}_i) $$
    *   Predict the position offset $\Delta \mu_i$:
        $$ \Delta \mu_i = \text{Tanh}(\mathcal{D}_{pos}(\mathbf{f}_i)) \cdot v_{\text{size}} $$
    *   Update the Gaussian center:
        $$ \mu_i = \mu^{init}_i + \Delta \mu_i $$
    *   **Training Loop:** Track $\Delta \mu_i$ from the current iteration as $\Delta \mu^{prev}_i$ for the next iteration. This recursive formulation encourages the network to query features at increasingly accurate positions near the object surface [S2].
    *   **Inference:** Initialize $\Delta \mu^{prev}_i = 0$. Recursively execute the prediction steps (query $\mathbf{f}_i$, predict $\Delta \mu_i$, update $\mu_i$, then predict $\alpha_i, \Sigma_i$) twice. The converged $\mu_i$ is used to query $\alpha_i$ and $\Sigma_i$ [S2].

5.  **Color Prediction:**
    *   Project the 3D Gaussians to the $K$ nearest reference views to retrieve 2D context, including color windows $\{c_k\}_{k=1}^K$ and visibility maps [S2].
    *   Decode the Gaussian color $c_i$ using the retrieved 2D context [S2].
    *   For background (distant views/sky), use a generalizable hemisphere Gaussian model with fixed geometry and predicted color [S2].

6.  **Rendering:**
    *   Compose foreground and background Gaussians.
    *   Render the image using alpha blending:
        $$ c = \sum_{i \in \mathcal{G}} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
        where $c$ is the rendered color, $\mathcal{G}$ is the set of Gaussians, and the product term represents transmittance [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def predict_gaussian_geometry(input_images, intrinsics, extrinsics, depth_estimator, sparse_3d_cnn, d_pos, d_opa, d_cov, v_size, num_inference_iters=2):
    """
    Predicts 3D Gaussian geometry parameters from input images.
    
    Args:
        input_images: List of N RGB images.
        intrinsics: List of N intrinsic matrices K_n.
        extrinsics: List of N extrinsic matrices T_n.
        depth_estimator: Function to predict depth maps.
        sparse_3d_cnn: Sparse 3D CNN to extract feature volume F.
        d_pos, d_opa, d_cov: MLP heads for position, opacity, covariance.
        v_size: Voxel size for scaling position offset.
        num_inference_iters: Number of recursive updates during inference.
        
    Returns:
        Gaussians: List of tuples (mu, alpha, sigma, color).
    """
    # 1. Depth Estimation and Point Cloud Construction
    depth_maps = [depth_estimator(img) for img in input_images]
    
    # Unproject to 3D
    raw_points = []
    for n in range(len(input_images)):
        I_n = input_images[n]
        D_n = depth_maps[n]
        K_n = intrinsics[n]
        T_n = extrinsics[n]
        # pi_inv performs pixel unprojection: (x, y, z) = K^{-1} * [u, v, 1] * depth
        points_n = unproject_3d(I_n, D_n, K_n, T_n)
        raw_points.append(points_n)
    
    P_raw = concatenate(raw_points)
    
    # 2. Filtering
    # Uniform filter to remove duplicates
    P_filtered = uniform_filter(P_raw)
    # Statistical filter to remove floaters
    P = statistical_filter(P_filtered)
    
    # 3. Feature Volume Construction
    # Construct sparse tensor from P
    sparse_tensor = construct_sparse_tensor(P)
    # Extract feature volume F
    F = sparse_3d_cnn(sparse_tensor)
    
    # 4. Initialization
    N_p = len(P)
    gaussians = []
    
    for i in range(N_p):
        mu_init = P[i]
        delta_mu_prev = np.zeros(3)
        
        # 5. Geometry Prediction (Recursive for convergence)
        # During training, delta_mu_prev is updated iteratively.
        # During inference, we run a fixed number of iterations.
        current_mu = mu_init
        
        for iter in range(num_inference_iters):
            # Query feature volume at current estimated position
            # Trilinear interpolation of F at (mu_init + delta_mu_prev)
            f_i = trilinear_interpolate(F, mu_init + delta_mu_prev)
            
            # Predict opacity and covariance
            alpha_i = d_opa(f_i)
            sigma_i = d_cov(f_i)
            
            # Predict position offset
            # Tanh activation followed by scaling with voxel size
            delta_mu_i = tanh(d_pos(f_i)) * v_size
            
            # Update position
            current_mu = mu_init + delta_mu_i
            
            # Update previous offset for next iteration (or final tracking)
            delta_mu_prev = delta_mu_i
            
        # Final query for alpha and sigma using converged position
        # Note: In the paper, alpha and sigma are predicted from the feature at the converged point
        f_final = trilinear_interpolate(F, current_mu)
        alpha_final = d_opa(f_final)
        sigma_final = d_cov(f_final)
        
        # 6. Color Prediction (Simplified placeholder for context retrieval)
        # In practice, project to nearest K views, retrieve color windows, and decode
        color_i = predict_color_from_context(current_mu, sigma_i, input_images, intrinsics, K=3)
        
        gaussians.append((current_mu, alpha_final, sigma_final, color_i))
        
    return gaussians

def unproject_3d(image, depth_map, K, T):
    """
    Unprojects 2D pixels to 3D points using depth, intrinsics, and extrinsics.
    """
    H, W = image.shape[:2]
    points = []
    for u in range(W):
        for v in range(H):
            depth = depth_map[v, u]
            if depth > 0:
                # Pixel to camera coordinates
                x_cam = (u - K[0, 2]) * depth / K[0, 0]
                y_cam = (v - K[1, 2]) * depth / K[1, 1]
                z_cam = depth
                point_cam = np.array([x_cam, y_cam, z_cam, 1.0])
                # Camera to world coordinates
                point_world = T @ point_cam
                points.append(point_world[:3])
    return np.array(points)

def trilinear_interpolate(volume, point):
    """
    Performs trilinear interpolation on a 3D volume at a given point.
    """
    # Normalize point to volume coordinates if necessary
    # Then perform trilinear interpolation
    # This is a standard mathematical operation
    return interpolate(volume, point)

def predict_color_from_context(mu, sigma, images, intrinsics, K=3):
    """
    Retrieves 2D context from nearest reference views to predict color.
    """
    # Project mu to each image plane
    # Retrieve color windows and visibility maps
    # Decode color using the context
    # This step is complex and depends on the specific implementation of the color decoder
    # Placeholder for the actual color prediction logic
    return np.array([0.5, 0.5, 0.5])
```

## Subtopics

### [Textured-GS](9436-textured-gs.md)
Textured-GS enhances the textural representation of individual 3D Gaussians by modeling spatially defined color and opacity variations across the Gaussian ellipsoidal surface using Spherical Harmonics (SH), without increasing the number of Gaussians or adding extra parameters beyond the existing SH framework. Unlike standard 3D Gaussian Splatting (3DGS) where each Gaussian retains a fixed opacity and shows a single color when viewed from a specific direction, Textured-GS enables color variation across the Gaussian ellipsoidal surface and opacity variations, allowing for deviations from the standard ellipsoidal shape such as 'unidirectional Gaussians' that are opaque on one side and transparent on the other. It differs from Gaussian Geometry Parameters Prediction by using the surface intersection vector for SH evaluation instead of the viewing vector, allowing spatial variation of color/opacity within a single Gaussian.

### [Mesh-Embedded Gaussian Splatting](9553-mesh-embedded-gaussian-splatting.md)
Mesh-Embedded Gaussian Splatting is a hybrid 3D representation that disentangels motion and appearance by embedding 3D Gaussians onto a canonical triangle mesh (such as SMPL-X or FLAME). Unlike methods relying on MLP-based linear blend skinning (LBS) fields for motion, this approach controls Gaussian rotation and translation directly via the mesh geometry, enabling compatibility with skeletal animation, blend shapes, and mesh editing. It builds on Gaussian Geometry Parameters Prediction by providing a structured, mesh-driven initialization and parameterization scheme.

### [Mesh-Embedded Gaussian Initialization and Deformation](9559-mesh-embedded-gaussian-initialization-and-deformation.md)
This method anchors 3D Gaussians to a triangle mesh using barycentric coordinates and normal displacement, enabling deformation tracking through mesh topology rather than direct vertex positions. It differs from standard Gaussian Splatting by decoupling Gaussian positions from fixed world coordinates, allowing them to deform with the underlying mesh, and from Textured-GS by using explicit geometric embeddings rather than texture maps for spatial definition.

### [Oriented Gaussian Splatting](9621-oriented-gaussian-splatting.md)
Oriented Gaussian Splatting is a variant of 3D Gaussian Splatting where each primitive is assigned a learnable oriented normal vector $n_i$ to create a reciprocal attenuation model, enabling the derivation of explicit geometric fields (normal and occupancy) via the Objects as Volumes framework. It differs from standard Gaussian Splatting by restoring reciprocity ($\sigma(x, w) = \sigma(x, -w)$), which is broken by standard attenuation, thereby allowing the extraction of watertight meshes and consistent normals.

### [3D Skew-Normal Splatting](10045-3d-skew-normal-splatting.md)
3D Skew-Normal Splatting (SNS) is a volume rendering method that replaces the standard symmetric 3D Gaussian kernel with the Azzalini Skew-Normal distribution to model asymmetric structures like sharp boundaries and one-sided surfaces. It sets itself apart from symmetric variants like 3D Gaussian Splatting and other asymmetric approaches like Oriented Gaussian Splatting by providing continuous shape interpolation between symmetric Gaussians and half-ellipsoids while preserving analytical tractability under affine transformations. This allows for seamless integration into existing splat-based rasterization pipelines without the distributional discontinuities introduced by hard truncation methods.

### [Deformable 3D Gaussian Splatting](10060-deformable-3d-gaussian-splatting.md)
Deformable 3D Gaussian Splatting is a method for dynamic scene reconstruction that learns 3D Gaussians in a canonical (static) space and models spatial-temporal deformation using an implicit deformation field implemented as a Multi-Layer Perceptron (MLP). It differs from static 3D Gaussian Splatting by introducing time-dependent offsets to Gaussian attributes, and from other dynamic variants like Mesh-Embedded Gaussian Splatting by relying on an implicit MLP rather than explicit mesh constraints or voxel grids.

### [Fourier Splatting Primitive](11094-fourier-splatting-primitive.md)
The Fourier Splatting primitive is a planar surfel for radiance field rendering that replaces the fixed Gaussian or elliptical falloff with a boundary defined by a Fourier series, allowing the representation of arbitrary closed shapes. Unlike 3D Gaussian Splatting (3DGS) or 2D Gaussian Splatting (2DGS), which are limited to fixed elliptical/disc shapes, this primitive enables higher fidelity representation of complex geometry and fine-grained details. Its key differentiator is inherent scalability: the level of detail (LoD) is controlled by truncating Fourier coefficients at runtime, allowing a single trained model to be rendered at varying complexities without retraining or post-hoc compression.

### [Conditional Slicing Mechanism](11455-conditional-slicing-mechanism.md)
The Conditional Slicing Mechanism conditions a 7-dimensional Gaussian (defined over spatial, temporal, and angular dimensions) on observed time $t$ and view direction $d$ to yield a conditional 3D Gaussian governing the spatial component. This operation "slices" the full 7D distribution to integrate temporal dynamics and view-dependent effects into the traditional 3D rendering pipeline without introducing additional temporal dependencies in the spherical harmonics. It sets itself apart from static Gaussian Splatting variants by explicitly modeling the joint distribution of space, time, and view, allowing the spatial mean and covariance to shift based on the current rendering context.

### [Structure-Aware 3D Gaussian Splatting (SAGS)](11908-structure-aware-3d-gaussian-splatting-sags.md)
Structure-Aware 3D Gaussian Splatting (SAGS) is a novel-view synthesis method that implicitly encodes scene geometry using a local-global graph representation to facilitate learning complex scenes and enforce meaningful point displacements that preserve structure, thereby reducing floating-point artifacts and storage requirements. Unlike standard 3D Gaussian Splatting which optimizes each Gaussian independently, SAGS leverages inter- and intra-connectivity between 3D point positions via Graph Neural Networks (GNNs) to predict Gaussian attributes, outperforming 3D-GS and Scaffold-GS in capturing high-frequency details while significantly reducing memory footprint.

### GSGEN
GSGEN introduces a 3D SDS loss based on Point-E to mitigate the multi-face or Janus problem in text-to-3D generation [S1]. It adopts Point-E to initialize the point cloud as the initial geometry for optimization and refines the appearance with only the 2D image prior [S1].

### BrightDreamer
BrightDreamer predicts the deviation of a set of fixed anchor points to determine the centers of Gaussians [S1]. The centers of Gaussians are determined from these deviations [S1].

### NeRF-Guided 3D Gaussian Splatting
NeRF-Guided 3D Gaussian Splatting uses a pretrained NeRF as a teacher to guide feature distillation, improving alignment between geometry and semantics [S3, unofficial].

### Gaussian Splatting Attribute Prediction
Gaussian Splatting Attribute Prediction is a voxel transformer that predicts residuals added to initial Gaussian rendering attributes on voxels, rather than generating new attributes directly [S4]. The predicted residual $\Delta \mathbf{G}_j$ is calculated as $\Delta \mathbf{G}_j = \text{MLP}_{fine}([\mathbf{O}_j, \mathbf{E}_j])$, where $\mathbf{O}_j$ is the offset and $\mathbf{E}_j$ is the voxel attribute [S4]. The final splat rendering parameter is $\mathbf{G}'_j = \mathbf{G}_j + \Delta \mathbf{G}_j$ [S4].

### 6D Gaussian Splatting
6D Gaussian Splatting is an extension of 3DGS that models view-dependent effects by adding directional dimensions, representing scene elements as 6D Gaussians (space+direction) [S5]. It parameterizes the full 6D covariance via Cholesky decomposition $\Sigma = LL^\top$ with $L$ lower-triangular and positive diagonal entries to guarantee positive definiteness [S5].

### Anisotropic Covariance
Anisotropic Covariance allows 3D Gaussians to scale anisotropically along the x, y, and z axes and rotate in 3D space, rather than remaining spherical, to better fit scene geometry [S6, unofficial]. Each Gaussian optimizes 3 scale parameters (x, y, z expansion) and 4 rotation parameters (quaternion) [S6, unofficial]. On Average-30K, Full achieves PSNR 26.05, while the Isotropic variant achieves 25.23 [S6, unofficial]. Isotropic forces Gaussians to remain spherical, causing blurry, rounded artifacts (e.g., tree trunks/leaves appear as round dots), whereas Anisotropic allows elongated/rotated shapes matching scene geometry [S6, unofficial].

### Defocus Blur Modeling
Defocus Blur Modeling models optical defocus by adjusting the scale $s$ of 3D Gaussians based on their distance from the camera's focal plane [S7]. During training, larger scale values $\bar{s}_j$ are used to rasterize blurred images, while during testing, smaller scale values $s_j$ are used to render sharp details [S7]. Unlike a naive Gaussian kernel that blurs the entire image uniformly, this method selectively enlarges covariances only for Gaussians corresponding to blurred regions [S7].

## Operating characteristics
*   **Time Complexity:** The method is feed-forward, enabling real-time rendering once the model is trained. The prediction of geometry parameters involves querying a sparse 3D CNN and three MLP heads per Gaussian, which is computationally efficient compared to per-scene optimization [S2].
*   **Memory Complexity:** Memory usage is dominated by the feature volume $F$ and the MLP parameters. The downsampling of initial primitives reduces the number of Gaussians, lowering memory consumption compared to dense per-pixel methods [S2].
*   **Measured Performance:** EVolSplat achieves real-time rendering speeds for urban scene reconstruction [S2]. It improves global consistency and reduces ghosting artifacts compared to pixel-aligned methods like MVSplat, Flash3D, and DrivingForward [S2].
*   **Scaling:** The method scales to urban scenes with large camera movements and small parallax, where traditional methods struggle due to the need for overlapping input images [S2].
*   **Strengths:** Global consistency, reduced ghosting artifacts, real-time rendering, efficient handling of foreground and background [S2].
*   **Limitations:** Relies on the quality of monocular depth estimators, which may contain noise and floating artifacts [S2]. The method is designed for urban scenes and may require adaptation for other scene types [S2].
*   **When to use:** Efficient feed-forward reconstruction of urban scenes from sparse views.
*   **When not to use:** Scenes requiring per-pixel local consistency without global volume accumulation, or scenes where high-quality monocular depth is unavailable.

## Background
Gaussian Geometry Parameters Prediction, specifically in the form of EVolSplat, was introduced by Sheng Miao, Jiaxin Huang, Dongfeng Bai, Xu Yan, Hongyu Zhou, Yue Wang, Bingbing Liu, Andreas Geiger, and Yiyi Liao from Zhejiang University, Huawei Noah’s Ark Lab, and the University of Tübingen [S2]. The paper "EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis" was published in CVPR 2025 [S2]. The method addresses the limitations of existing feed-forward 3DGS methods that suffer from multi-view inconsistencies and duplicated content by predicting a global scene representation [S2].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S4] Learning Efficient Fuse-and-Refine for Feed-Forward 3D Gaussian Splatting, https://arxiv.org/abs/2503.14698v2, fetched 2026-10-27
[S5] 7DGS: Unified Spatial-Temporal-Angular Gaussian Splatting, https://arxiv.org/abs/2503.07946v1, fetched 2026-10-26
[S6] GaussianSplatting論文のAblationについての解説・考察, https://qiita.com/kotauchisunsun/items/f1e89f28aa770b8d3b19, fetched 2026-10-29 (unofficial)
[S7] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
