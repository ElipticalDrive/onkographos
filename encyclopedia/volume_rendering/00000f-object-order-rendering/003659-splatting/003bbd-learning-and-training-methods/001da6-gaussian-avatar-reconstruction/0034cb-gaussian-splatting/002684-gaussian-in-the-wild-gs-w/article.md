# Gaussian in the Wild (GS-W)
Also called: GS-W, Intrinsic and Dynamic Appearance Fusion

## Overview
Gaussian in the Wild (GS-W) is a novel view synthesis method for unconstrained image collections that explicitly separates scene appearance into intrinsic material properties and dynamic environmental factors (such as illumination and weather) per 3D Gaussian point [S2]. It differs from standard 3D Gaussian Splatting and NeRF-based methods by avoiding global latent embeddings, which allows it to capture local high-frequency changes without blurring intrinsic textures with dynamic variations [S2]. The method employs adaptive sampling from image features and 2D visibility maps to mitigate transient occlusions and photometric variations [S1][S3, unofficial].

## How it works
### Inputs and Outputs
- **Inputs**: A set of unconstrained 2D images with known camera poses (extrinsics and intrinsics).
- **Outputs**: A rendered image from a novel viewpoint, and implicitly, the optimized 3D Gaussian scene representation.

### Data Structures
The scene is represented as a collection of $M$ 3D Gaussian points $GP$. Each Gaussian point $i$ stores the following attributes:
1.  **Position** $X_i \in \mathbb{R}^3$: The center of the Gaussian.
2.  **Covariance** $\boldsymbol{\Sigma}_i \in \mathbb{R}^{3 \times 3}$: Modeled via a scaling matrix $S$ and rotation matrix $R$ (stored as quaternion $q$) to ensure positive semi-definiteness during optimization [S2].
    -   $\boldsymbol{\Sigma} = R S S^T R^T$ [S2].
3.  **Intrinsic Appearance Feature** $s_{f}^i \in \mathbb{R}^d$: A learnable vector representing the static material properties of the point [S2].
4.  **Dynamic Appearance Feature** $d_{f}^i \in \mathbb{R}^d$: A feature vector sampled from image features, representing dynamic environmental factors [S2].
5.  **Opacity** $\alpha_i$: Controls transparency [S2].
6.  **View-dependent Color**: Derived from the fused appearance feature and view direction via an MLP [S2].

### Rendering Pipeline
1.  **Projection**: Each 3D Gaussian $i$ is projected onto the 2D image plane. The 3D covariance $\boldsymbol{\Sigma}_i$ is transformed into the 2D screen space covariance $\boldsymbol{\Sigma}'_i$ using the affine approximation of the projective transformation $J$ and the view matrix $W$ [S2]:
    $$ \boldsymbol{\Sigma}'_i = J W \boldsymbol{\Sigma}_i W^T J^T $$
    The 2D Gaussian function is defined as:
    $$ G(x - X, \boldsymbol{\Sigma}) = e^{-\frac{1}{2}(x-X)^T \boldsymbol{\Sigma}^{-1}(x-X)} $$
    where $x$ is a point in space (or pixel coordinates) and $X$ is the Gaussian center [S2].

2.  **Sorting**: Gaussian points are sorted by depth for each tile on the image plane. The image plane is divided into a grid of $16 \times 16$ tiles [S2].

3.  **Alpha Blending**: For each pixel $px'$, the color $C(px')$ is aggregated from the sorted Gaussian points $GP_{px'}$ using alpha blending [S2]:
    $$ \sigma_i = G(px' - X_i, \boldsymbol{\Sigma}'_i) $$
    $$ C(px') = \sum_{i \in GP_{px'}} c_i \sigma_i \prod_{j=1}^{i-1} (1 - \sigma_j) $$
    where $c_i$ is the color of the $i$-th Gaussian point [S2].

### Appearance Modeling
The core innovation of GS-W is the separation and fusion of appearance features.

1.  **Feature Extraction**: Image features are extracted from a reference image using a U-Net model. These features are reshaped into $K$ feature maps and one projection feature map [S2].

2.  **Dynamic Feature Sampling**: For each Gaussian point $GP_i$, dynamic appearance features $d_{f}^i$ are sampled from the extracted feature maps using an **adaptive sampling strategy**. This strategy allows each point to focus on local and detailed information more effectively [S2].

3.  **Fusion**: The intrinsic appearance feature $s_{f}^i$ (learnable per point) and the sampled dynamic appearance feature $d_{f}^i$ are fused with the position $X_i$ using a fusion network $M_f$. The network $M_f$ is composed of two Multi-Layer Perceptrons (MLPs) [S2]:
    $$ a_{f}^i = M_f(s_{f}^i, d_{f}^i, X_i) $$
    where $a_{f}^i$ is the holistic appearance feature [S2].

4.  **Color Decoding**: The fused appearance feature $a_{f}^i$ and the view direction $\phi$ are decoded by another MLP $M_c$ to obtain the final color $c_i$ of the Gaussian point [S2]:
    $$ c_i = M_c(a_{f}^i, \phi) $$

### Training and Optimization
1.  **Initialization**: 3D Gaussian points $GP$ are initialized with positions $X$ extracted from a point cloud generated via Structure-from-Motion (SfM) from the input image set [S2].

2.  **Loss Calculation**: The loss is computed between the rendered image $I_r$ and the ground truth reference image $I_{gt}$. To mitigate the impact of transient occluders (e.g., moving people, vehicles), a **2D visibility map** $V_M \in \mathbb{R}^{1 \times H \times W}$ is employed. This map is obtained from a U-Net model and facilitates segmentation between transient and static objects. The loss is weighted by this visibility map [S2][S3, unofficial].

3.  **Optimization**: All Gaussian attributes (position, covariance, opacity, intrinsic features $s_f^i$) are jointly optimized. The method employs point growth and pruning strategies based on gradients and opacity to manage the number of Gaussians [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gs_w_render(reference_images, camera_poses, initial_point_cloud):
    """
    Main pipeline for Gaussian in the Wild (GS-W).
    """
    # 1. Initialization
    # Extract points via SfM
    gaussians = initialize_gaussians(initial_point_cloud)
    
    # Initialize learnable intrinsic appearance features for each Gaussian
    for g in gaussians:
        g.intrinsic_feature = learnable_vector()
        
    # Initialize networks
    fusion_net = FusionNetwork() # Two MLPs
    color_net = ColorNetwork()   # MLP
    feature_extractor = UNet()   # For image features and visibility map
    
    # 2. Training Loop
    for epoch in range(num_epochs):
        for ref_img, cam_pose in zip(reference_images, camera_poses):
            
            # A. Feature Extraction
            # Extract features from reference image
            img_features = feature_extractor(ref_img)
            # Reshape into K feature maps and one projection feature map
            feature_maps = reshape_features(img_features)
            
            # Extract 2D visibility map to handle transient objects
            visibility_map = feature_extractor.get_visibility_map(ref_img)
            
            # B. Dynamic Feature Sampling
            for g in gaussians:
                # Adaptive sampling from feature maps based on Gaussian position
                g.dynamic_feature = adaptive_sample(feature_maps, g.position)
                
            # C. Appearance Fusion and Color Prediction
            for g in gaussians:
                # Fuse intrinsic, dynamic, and position
                holistic_feature = fusion_net(
                    g.intrinsic_feature, 
                    g.dynamic_feature, 
                    g.position
                )
                
                # Decode color based on holistic feature and view direction
                # phi is the view direction from camera to Gaussian center
                view_direction = normalize(cam_pose.camera_center - g.position)
                g.color = color_net(holistic_feature, view_direction)
                
            # D. Rasterization
            # Project Gaussians to 2D screen space
            projected_gaussians = []
            for g in gaussians:
                # Transform 3D covariance to 2D
                # J: affine approximation of projective transformation
                # W: view matrix
                sigma_2d = project_covariance(g.covariance, cam_pose, J, W)
                projected_gaussians.append({
                    'position': project_point(g.position, cam_pose),
                    'covariance': sigma_2d,
                    'color': g.color,
                    'opacity': g.opacity
                })
            
            # Sort by depth and render using tile-based alpha blending
            projected_gaussians.sort_by_depth()
            rendered_image = tile_rasterize(projected_gaussians, image_width, image_height)
            
            # E. Loss Computation
            # Weight loss by visibility map to ignore transient objects
            loss = weighted_mse_loss(rendered_image, ref_img, visibility_map)
            
            # F. Optimization Step
            loss.backward()
            optimizer.step()
            
            # G. Point Management
            prune_and_grow_gaussians(gaussians, gradients, opacity_threshold)

    return gaussians
```

### Edge Cases and Pitfalls
-   **Transient Occluders**: Without the 2D visibility map, moving objects (people, cars) cause artifacts like floating points or ghosting because the model attempts to reconstruct their geometry and appearance as static [S2][S3, unofficial].
-   **Dynamic Variations**: Failing to separate intrinsic and dynamic features leads to blurring of material textures when lighting or weather changes, as the model conflates static properties with environmental effects [S2].
-   **Local Detail**: Global appearance representations (common in NeRF) struggle with high-frequency local changes. GS-W's per-point adaptive sampling mitigates this [S2].

## Operating characteristics
-   **Rendering Speed**: GS-W surpasses state-of-the-art NeRF-based methods in rendering speed by over 1000x [S2]. It matches real-time rendering speeds similar to standard 3D Gaussian Splatting (3DGS) on wild data [S1].
-   **Quality**: Outperforms NeRF-based methods in reconstruction quality and detail, particularly in unconstrained environments with dynamic lighting and transient objects [S2].
-   **Memory**: Requires storage for intrinsic features per Gaussian point, increasing memory usage compared to standard 3DGS which typically uses a single SH coefficient set or simple color per point. The exact memory overhead is not specified in the sources.
-   **Strengths**: Handles unconstrained image collections (wild data) with varying illumination, weather, and transient occluders. Real-time rendering capability.
-   **Limitations**: Requires a U-Net for feature extraction and visibility map generation, adding complexity to the training pipeline. Performance depends on the quality of the SfM initialization and the visibility map accuracy.

## Background
-   **Origin**: Gaussian in the Wild (GS-W) was proposed to address the limitations of existing NeRF-based methods and standard 3DGS when applied to unconstrained image collections. Standard methods assume static scenes and consistent lighting, which is rarely the case in real-world "wild" data [S2].
-   **Key Insight**: The appearance of a point in the real world is determined by its intrinsic material properties and dynamic environmental impacts. Separating these allows for better reconstruction of static geometry and appearance while handling dynamic variations [S2].
-   **Development**: GS-W builds upon the 3D Gaussian Splatting framework [16] (referenced as [S2] in the context of 3DGS) and introduces the appearance separation module, adaptive sampling, and visibility map handling [S2]. It is cited in surveys as a key method for handling wild data alongside Wild-GS and SWAG [S1][S3, unofficial].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
[S2] Gaussian in the Wild: 3D Gaussian Splatting for Unconstrained Image Collections, https://doi.org/10.1007/978-3-031-73116-7_20, fetched 2026-10-27
[S3] From Volume Rendering to 3D Gaussian Splatting, https://www.visgraf.impa.br/Data/RefBib/PS_PDF/tutorial-sib2025/tutorial-sib2025.pdf, fetched 2026-10-28 (unofficial)
