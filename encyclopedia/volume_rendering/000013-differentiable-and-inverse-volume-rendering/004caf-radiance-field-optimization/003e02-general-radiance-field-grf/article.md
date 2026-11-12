# General Radiance Field (GRF)

## Overview
General Radiance Field (GRF) is a neural network-based method that implicitly represents and renders 3D scenes by learning local features for each pixel in 2D images and projecting them to 3D space [S1]. It constructs an internal representation for every point in 3D space, allowing the rendering of appearance and geometry from an arbitrary viewpoint [S1]. Unlike standard NeRF which maps 3D coordinates directly to radiance, GRF extracts features from 2D input images and aggregates them via reprojection, making the representation dependent on the input views rather than just spatial coordinates [S1].

## How it works
**Inputs:**
*   A set of $M$ input images with their corresponding camera poses and intrinsics: $\{(I_1, \xi_1, K_1), \dots, (I_M, \xi_M, K_M)\}$ [S1].
*   A query 3D point $p = \{x_p, y_p, z_p\}$ [S1].
*   A query viewpoint $V_p = \{x_{vp}, y_{pv}, z_{pv}\}$ [S1].

**Outputs:**
*   Predicted RGB value $\{r_p, g_p, b_p\}$ and volumetric density $d_p$ for point $p$ observed from viewpoint $V_p$ [S1].

**Data Structures:**
The method utilizes a neural network architecture $f_W$ with learnable parameters $W$ [S1]. The network consists of four main components:
1.  **Feature Extractor:** A Convolutional Neural Network (CNN) encoder-decoder with skip connections [S1].
2.  **Reprojector:** Transforms 2D features to 3D space [S1].
3.  **Aggregator:** Obtains general features for 3D points by aggregating features from multiple views [S1].
4.  **Neural Renderer:** An MLP-based renderer (based on NeRF) that infers appearance and geometry from the aggregated features [S1].

**Formulas:**
The core prediction is defined as [S1]:
$$
(r_p, g_p, b_p, d_p) = f_W(I_1, \xi_1, K_1, \dots, I_M, \xi_M, K_M, x_p, y_p, z_p, x_{vp}, y_{pv}, z_{pv})
$$
where $(r_p, g_p, b_p, d_p)$ are the predicted RGB and density, and $f_W$ is the network [S1].

**Steps:**
1.  **Feature Extraction:** For each input image $I_m$, the CNN extracts features for every pixel. The input to the CNN is the raw RGB image with the camera viewpoint coordinates (xyz location) stacked/duplicated to each pixel [S1].
    *   *Note:* Stacking camera rotation and intrinsics does not noticeably improve performance [S1].
    *   Skip connections are used between encoder and decoder to preserve high-frequency local features [S1].
2.  **Reprojection:** The 2D features extracted from the image pixels are reprojected back to the query 3D point $p$ [S1]. This involves identifying which pixels in the input images "see" the point $p$ and retrieving their features.
3.  **Aggregation:** Features from multiple 2D views (pixels that project to $p$) are aggregated to form a general feature representation for the 3D point [S1]. The paper mentions integrating an attention mechanism to handle visual occlusions implicitly [S1].
4.  **Rendering:** The aggregated features for point $p$ and the viewpoint $V_p$ are fed into the neural renderer (MLP) to predict $\{r_p, g_p, b_p, d_p\}$ [S1].
5.  **Volume Rendering:** Multiple points along a light ray are integrated using the volume rendering equation to obtain the final rendered pixel RGB [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# GRF Forward Pass
def grf_forward(input_images, camera_poses, camera_intrinsics, query_points, query_viewpoints):
    """
    input_images: List of M images (H, W, 3)
    camera_poses: List of M camera positions (M, 3)
    camera_intrinsics: List of M camera intrinsics (M, 3, 3)
    query_points: Array of N 3D points (N, 3)
    query_viewpoints: Array of N 3D viewpoints (N, 3)
    """
    # 1. Feature Extraction
    # Stack viewpoint coordinates to each pixel
    # Input to CNN: (H, W, 3 + 3) = (H, W, 6)
    # Note: Rotation and intrinsics are NOT stacked per empirical findings [S1]
    pixel_features_list = []
    for m in range(M):
        img = input_images[m]
        cam_pos = camera_poses[m]
        # Duplicate cam_pos to match image dimensions
        cam_pos_grid = np.broadcast_to(cam_pos, (img.shape[0], img.shape[1], 3))
        img_with_pos = np.concatenate([img, cam_pos_grid], axis=-1)
        
        # Extract features using CNN with skip connections
        # Returns feature map for each pixel
        features = cnn_extractor(img_with_pos) 
        pixel_features_list.append(features)

    # 2. Reprojection and Aggregation
    # For each query point p, find corresponding pixels in input images
    # Project p into each image plane using camera poses/intrinsics
    # Retrieve features from pixel_features_list at those pixel locations
    
    aggregated_features = []
    for n in range(N):
        p = query_points[n]
        vp = query_viewpoints[n]
        
        view_features = []
        for m in range(M):
            # Project p to 2D pixel coordinates (u, v) in image m
            uv = project_point_to_image(p, camera_poses[m], camera_intrinsics[m])
            
            # Check if uv is within image bounds
            if is_valid_pixel(uv, input_images[m].shape):
                # Get feature vector at (u, v) from image m
                feat = pixel_features_list[m][uv.y, uv.x]
                view_features.append(feat)
        
        # Aggregate features from multiple views
        # The paper mentions an attention mechanism for aggregation [S1]
        # If no views see the point, handle appropriately (e.g., zero vector)
        agg_feat = aggregate_features(view_features) # Attention-based aggregation
        aggregated_features.append(agg_feat)

    # 3. Neural Rendering
    # Concatenate aggregated features with viewpoint info
    # Input to MLP: [aggregated_features, query_viewpoints]
    rendering_input = np.concatenate([np.array(aggregated_features), query_viewpoints], axis=-1)
    
    # Predict RGB and Density
    rgb_density = mlp_renderer(rendering_input)
    
    return rgb_density

# Volume Rendering Integration (Standard DVR)
def volume_render(ray_origins, ray_directions, sampled_points, rgb_density):
    """
    Integrate points along rays using the standard volume rendering equation.
    """
    # Compute transmittance and accumulated color
    # T_i = exp(-sum(sigma_j * delta_j))
    # C = sum(T_i * (1 - exp(-sigma_i * delta_i)) * c_i)
    # This step is standard and relies on the differentiability of the forward rendering operator [44-radiance-field-optimization.md]
    pass
```

**Parameters:**
*   **CNN Module:** Uses skip connections between encoder and decoder [S1]. Fully connected layers may be optionally integrated in the middle to learn global features [S1].
*   **Input Augmentation:** Camera viewpoint (xyz) is stacked to each pixel [S1]. Camera rotation and intrinsics are not stacked [S1].

**Edge Cases and Pitfalls:**
*   **Lighting Sensitivity:** Using raw RGB values as pixel features is sub-optimal because they are sensitive to lighting conditions and environmental noise [S1]. The CNN extractor mitigates this by learning robust patterns [S1].
*   **Camera Intrinsics:** Stacking additional camera rotation and intrinsics to each pixel does not noticeably improve performance [S1].
*   **Occlusion:** Visual occlusions are handled implicitly by the aggregation mechanism (attention) [S1].

## Subtopics

### Hierarchical Grid Optimization
Uses hierarchical data structures to accelerate feature lookup and aggregation [1414-hierarchical-grid-optimization.md].

### 3D Gaussian Splatting
Represents the scene as a set of 3D Gaussians instead of a neural field, offering different trade-offs in rendering speed and quality [1795-3d-gaussian-splatting.md].

### Neural Radiance Fields (NeRF)
The foundational method that GRF builds upon, mapping 3D coordinates directly to radiance without explicit 2D feature extraction [4972-neural-radiance-fields.md].

## Operating characteristics
*   **Memory:** The method requires storing features for all input images and the neural network parameters. The memory usage depends on the resolution of the input images and the size of the CNN and MLP [S1].
*   **Speed:** The speed is determined by the CNN feature extraction, the reprojection/aggregation step, and the MLP inference. The aggregation step involves projecting points to multiple images, which can be computationally intensive if not optimized [S1].
*   **Strengths:** The explicit use of 2D features allows for leveraging powerful 2D CNN architectures and attention mechanisms. It can handle complex lighting and texture variations better than raw RGB [S1].
*   **Limitations:** The method relies on the quality of the 2D feature extractor. If the CNN fails to extract robust features, the 3D representation will be poor. The reprojection step adds complexity compared to direct coordinate mapping [S1].

## Background
General Radiance Field (GRF) was introduced by Alex Trevithick and Bo Yang in 2021 [S1]. The work was presented at ICCV 2021 [S1]. It extends the concept of Neural Radiance Fields (NeRF) by incorporating 2D image features into the 3D representation learning process [S1].

## Sources
[S1] GRF: Learning a General Radiance Field for 3D Representation and Rendering, https://doi.org/10.1109/iccv48922.2021.01490, fetched 2026-10-26
