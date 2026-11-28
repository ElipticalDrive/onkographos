# Neural Rays (NeuRay)

Also called: NeuRay, NeuRay Generalization via Cost Volume/Depth CNN, Visibility Feature Map Generation

## Overview
NeuRay is a neural volume rendering method designed to generalize to unseen scenes by explicitly modeling view visibility through 2D feature maps derived from multi-view stereo data [S1, S2]. Unlike methods that rely solely on 3D coordinate queries for density and color, NeuRay constructs visibility feature maps $G$ from cost volumes or depth maps using a CNN, allowing the renderer to account for occlusions from input views during novel view synthesis [S1, S2]. It sets itself apart by decoupling the estimation of visibility and local features into a 2D image-space processing step before aggregating them along rays, enabling effective fine-tuning on target scenes [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of input views (images) with known camera poses. For training, ground truth images are used; for inference, only input views are required.
*   **Outputs:** A rendered novel view image.

**Data Structures:**
*   **Cost Volume:** A 3D tensor of size $H \times W \times D$, where $H, W$ are image dimensions and $D$ is the number of depth planes [S1].
*   **Depth Map:** A 2D tensor of size $H \times W$ containing estimated depth values [S1].
*   **Visibility Feature Map ($G$):** A 3D tensor of size $H \times W \times C$, where $C$ is the feature channel dimension [S1].
*   **Intermediate Feature Map ($G'$):** A 3D tensor of size $H \times W \times C$, used during fine-tuning [S1].

**Steps:**

1.  **Multi-View Stereo (MVS) Estimation:**
    For each input view, construct a cost volume using $N_s$ neighboring input views, or estimate a depth map using patch-match stereo [S1].
    *   *Cost Volume Construction:* Align features from $N_s$ neighboring views to the current view's coordinate system and aggregate them into a volume of size $H \times W \times D$ [S1].
    *   *Depth Estimation:* Use a patch-match stereo algorithm (e.g., COLMAP) to generate a dense depth map of size $H \times W$ [S1].

2.  **Visibility Feature Map Generation:**
    Process the cost volume or depth map with a CNN to produce the visibility feature map $G \in \mathbb{R}^{H \times W \times C}$ [S1].
    *   If using a cost volume, the CNN takes the $H \times W \times D$ volume as input.
    *   If using a depth map, the CNN takes the $H \times W$ depth map as input [S1].

3.  **Ray Sampling and Visibility Computation:**
    For a test ray, sample $N$ 3D points. For each point $j$ on the ray and each input view $i$, compute the visibility $v_{i,j}$ of view $i$ to point $j$ [S1].
    *   Extract local features $f_{i,j}$ from the visibility feature map $G$ of view $i$ corresponding to the 2D projection of point $j$ [S1].
    *   Compute visibility $v_{i,j}$ using a cumulative density function formulated as a weighted sum of sigmoid functions [S2]. The exact formula is not provided in the sources, but it relies on the features extracted from $G$ [S2].

4.  **Alpha and Color Prediction:**
    Aggregate the local features $f_{i,j}$ and visibility $v_{i,j}$ using MLPs to predict the alpha values $\alpha_i$ and colors $\mathbf{c}_i$ for each sample point [S1, S2].
    *   The aggregation network blends input colors and applies a transformer along test rays, using visibility as an additional input [S1].

5.  **Volume Rendering:**
    Compute the final pixel color using the standard front-to-back volume rendering equation [S2]:
    $$
    \mathbf{C}_{out} = \sum_{i=1}^{N} \mathbf{c}_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$

**Parameters:**
*   $D$: Number of depth planes in the cost volume. Typical value: 64 [S1].
*   $N_s$: Number of neighboring input views used for cost volume construction. Typical value: 8 (denoted as $N_w$ in implementation details) [S1].
*   $N$: Number of sample points per ray. Typical value: 64 per stage (coarse and fine) [S1].

**Edge Cases and Pitfalls:**
*   **Direct Trainability:** Making the final visibility map $G$ directly trainable without a visibility encoder yields worse results. The visibility encoder is necessary to associate feature vectors of nearby pixels, which usually have similar visibility [S1].
*   **Initialization:** The intermediate feature map $G'$ must be initialized by the initialization network; training from scratch is not recommended [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neu_ray_render(input_views, camera_poses, test_ray_origin, test_ray_direction):
    """
    Renders a novel view using NeuRay.
    
    Args:
        input_views: List of input images and their poses.
        test_ray_origin: Origin of the test ray.
        test_ray_direction: Direction of the test ray.
        
    Returns:
        Rendered pixel color.
    """
    # Step 1: MVS Estimation
    # Construct cost volumes or depth maps for all input views
    cost_volumes = []
    depth_maps = []
    for view in input_views:
        # Use Ns neighboring views to construct cost volume
        # Or use patch-match stereo to estimate depth
        # Implementation detail: COLMAP for patch-match stereo [S1]
        cv = construct_cost_volume(view, input_views, Ns=8)
        dm = estimate_depth_map(view, input_views)
        cost_volumes.append(cv)
        depth_maps.append(dm)
        
    # Step 2: Visibility Feature Map Generation
    # Process cost volumes/depth maps with CNN to get G
    visibility_maps = []
    for cv, dm in zip(cost_volumes, depth_maps):
        # CNN processes either cv or dm
        # Output: G in R^(H x W x C)
        G = cnn_process_visibility(cv, dm)
        visibility_maps.append(G)
        
    # Step 3: Ray Sampling and Visibility Computation
    # Sample points along the test ray
    sample_points = sample_points_along_ray(test_ray_origin, test_ray_direction, N=64)
    
    # For each sample point and each input view, compute visibility and features
    visibilities = []
    local_features = []
    for i, view in enumerate(input_views):
        view_visibilities = []
        view_features = []
        for point in sample_points:
            # Project point to view i's image plane
            u, v = project_point_to_image(point, camera_poses[i])
            
            # Extract local features from G
            f_ij = extract_feature(visibility_maps[i], u, v)
            view_features.append(f_ij)
            
            # Compute visibility v_i,j
            # Formula: cumulative density function as weighted sum of sigmoids [S2]
            # Exact formula not provided in sources, assumed to be a function of f_ij
            v_ij = compute_visibility_from_features(f_ij)
            view_visibilities.append(v_ij)
            
        visibilities.append(view_visibilities)
        local_features.append(view_features)
        
    # Step 4: Alpha and Color Prediction
    # Aggregate features and visibilities using MLPs
    alphas = []
    colors = []
    for j, point in enumerate(sample_points):
        # Aggregate features from all views for this point
        aggregated_feature = aggregate_features(local_features[:, j], visibilities[:, j])
        
        # Predict alpha and color
        alpha_j, color_j = predict_alpha_color(aggregated_feature)
        alphas.append(alpha_j)
        colors.append(color_j)
        
    # Step 5: Volume Rendering
    # Front-to-back rendering
    color = 0.0
    accumulated_alpha = 1.0
    for j in range(len(sample_points)):
        alpha = alphas[j]
        color_j = colors[j]
        color += color_j * alpha * accumulated_alpha
        accumulated_alpha *= (1 - alpha)
        
    return color
```

## Subtopics

### NeuRay Finetuning with Trainable Intermediate Features
A finetuning method where an intermediate feature map $G' \in \mathbb{R}^{H \times W \times C}$ between the cost volume/depth and the final visibility map $G$ is treated as trainable parameters [S1]. $G'$ is initialized by an initialization network that processes the cost volume or depth map, and the initialization network is discarded during finetuning [S1]. $G'$ and the visibility encoder parameters are optimized using a consistency loss and rendering loss [S1]. This approach improves visibility prediction because the visibility encoder associates feature vectors of nearby pixels, which usually have similar visibility [S1]. Directly making $G$ trainable without the visibility encoder yields worse results [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Experiments conducted on a 2080 Ti GPU [S1]. NeuRay generalizes well to new scenes and can exceed the performance of baseline NeRF models when fine-tuned [S2].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Explicitly models visibility, allowing for better handling of occlusions in unseen scenes. Fine-tuning capability allows adaptation to specific scenes [S1, S2].
*   **Limitations:** Requires multi-view stereo data (cost volumes or depth maps) as input, which adds preprocessing steps [S1].
*   **When to use:** When rendering novel views for unseen scenes where occlusion handling is critical and multi-view input data is available [S1, S2].
*   **When not to use:** When single-view input is available or when MVS preprocessing is not feasible [S1].

## Background
NeuRay was introduced by Liu et al. in July 2021 [S2]. It is designed for occlusion-aware image-based rendering in unseen scenes [S1]. The method builds on the principles of neural volume rendering, specifically leveraging cost volumes and CNNs to extract visibility features [S1, S2]. It has been compared with baselines such as NeRF and pixelNeRF [S2].

## Sources
[S1] Neural Rays for Occlusion-aware Image-based Rendering, https://doi.org/10.1109/cvpr52688.2022.00767, fetched 2026-10-26
[S2] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
