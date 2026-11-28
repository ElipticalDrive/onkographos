# Feature Grid 3D Representation

Also called: Continuous Volumetric Field with Features

## Overview
The Feature Grid 3D Representation is a neural scene representation that extends the original NeRF framework by replacing the direct RGB color output with a multi-channel feature vector $F(\mathbf{x}) \in \mathbb{R}^C$ at every queried 3D position $\mathbf{x}$ [S1]. It solves the problem of enabling high-level semantic or stylistic processing of 3D scenes by decoupling geometric appearance (density) from high-dimensional feature encoding [S1]. Unlike standard NeRF which outputs RGB directly, this method renders full-resolution feature maps via volume integration, which are subsequently mapped to RGB space using a 2D CNN decoder [S1]. This approach discards intermediate up-sampling operations, thereby eliminating multi-view inconsistency and aliasing artifacts common in stylized video rendering [S1].

## How it works
**Inputs:**
*   A set of input images with known camera poses.
*   A reference style image (for style transfer variants) or ground-truth RGB images (for reconstruction).
*   Sampling points along rays passing through the scene.

**Outputs:**
*   Rendered RGB images.
*   Volume density field $\sigma(\mathbf{x})$.
*   Multi-channel feature field $F(\mathbf{x}) \in \mathbb{R}^C$.

**Data Structures:**
The representation utilizes two separate grids stored in memory:
1.  **Density Grid:** Stores the scalar volume density $\sigma(\mathbf{x})$ for each queried position.
2.  **Feature Grid:** Stores the multi-channel feature vector $F(\mathbf{x}) \in \mathbb{R}^C$ for each queried position.

To manage memory complexity, the Feature Grid employs **vector-matrix tensor decomposition** [S1]. A standard 3D voxel grid storing features has a space complexity of $O(n^3)$, where $n$ is the grid resolution [S1]. The decomposition relaxes low-rank constraints for two modes of the 3D tensor, factorizing it into compact vector and matrix factors [S1]. This reduces the space complexity to $O(n^2)$, massively reducing the memory footprint [S1].

**Parameters:**
*   $C$: The number of feature channels. Typical values are not specified in the source, but the grid stores $C$ channels per voxel [S1].
*   $N$: The number of sample points along a ray.

**Algorithm Steps:**
1.  **Grid Initialization:** Initialize a density grid and a feature grid using vector-matrix tensor decomposition to store $\sigma(\mathbf{x})$ and $F(\mathbf{x})$ respectively [S1].
2.  **Ray Sampling:** For a given camera ray $\mathbf{r}$, sample $N$ points $\mathbf{x}_i$ along the ray segment within the scene bounds [S1].
3.  **Feature Query:** For each sampled point $\mathbf{x}_i$, query the density grid to obtain $\sigma_i$ and the feature grid to obtain the feature vector $F_i$ [S1].
4.  **Volume Rendering of Features:** Compute the aggregated feature $F(\mathbf{r})$ for the ray by integrating the sampled features weighted by their transmittance [S1]. The weight $w_i$ for point $i$ is calculated as:
    $$ w_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) (1 - \exp(-\sigma_i \delta_i)) $$
    where $\delta_i$ is the distance between adjacent samples [S1]. The ray feature is then:
    $$ F(\mathbf{r}) = \sum_{i=1}^{N} w_i F_i $$
    [S1].
5.  **View-Dependency Handling:** Disable view-dependent effects in the feature grid to ensure multi-view consistency [S1].
6.  **Feature Map Generation:** Render full-resolution feature maps corresponding to the final RGB image resolution, rather than down-sampled maps [S1].
7.  **Color Decoding:** Pass the rendered feature maps through a 2D CNN decoder to map the high-level features to RGB space [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_feature_grid_ray(ray_origin, ray_direction, density_grid, feature_grid, num_samples):
    """
    Renders the feature vector for a single ray using the Feature Grid representation.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        density_grid: Function/Structure to query volume density sigma(x).
        feature_grid: Function/Structure to query feature vector F(x) of dimension C.
        num_samples: Integer, number of points to sample along the ray.
        
    Returns:
        F_ray: Feature vector of dimension C for the ray.
    """
    # Sample points along the ray
    # Assume t values are generated linearly or via stratified sampling
    t_values = generate_t_values(num_samples) 
    
    points = []
    for t in t_values:
        point = ray_origin + t * ray_direction
        points.append(point)
        
    densities = []
    features = []
    deltas = []
    
    # Query grids and compute deltas
    for i in range(len(points)):
        sigma = density_grid.query(points[i])
        F = feature_grid.query(points[i])
        densities.append(sigma)
        features.append(F)
        
        if i > 0:
            delta = distance(points[i], points[i-1])
        else:
            delta = 0.0 # First delta is typically handled by the exponential term logic or set to 0
        deltas.append(delta)
        
    # Compute weights and aggregate features
    # w_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j) * (1 - exp(-sigma_i * delta_i))
    # Note: The sum in the exponent is over previous samples.
    
    accumulated_transmittance = 1.0
    F_ray = zeros(C) # Initialize feature vector of size C
    
    for i in range(num_samples):
        sigma_i = densities[i]
        F_i = features[i]
        delta_i = deltas[i]
        
        # Calculate weight w_i
        # The term exp(-sum_{j=1}^{i-1} sigma_j * delta_j) is the accumulated transmittance before sample i
        # We update accumulated_transmittance after calculating w_i for the next iteration
        
        # However, the formula defines w_i using the sum up to i-1.
        # Let T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
        
        weight = accumulated_transmittance * (1 - exp(-sigma_i * delta_i))
        
        # Add weighted feature to the ray feature
        F_ray += weight * F_i
        
        # Update accumulated transmittance for the next sample
        accumulated_transmittance *= exp(-sigma_i * delta_i)
        
    return F_ray

def render_full_resolution_feature_map(camera_params, scene_model):
    """
    Renders full-resolution feature maps for a view.
    
    Args:
        camera_params: Camera pose and intrinsics.
        scene_model: Contains density_grid, feature_grid, and cnn_decoder.
        
    Returns:
        RGB_image: Rendered RGB image.
    """
    # Generate ray directions for every pixel in the image
    rays = generate_rays_from_camera(camera_params)
    
    # Render feature map at full resolution (same as RGB image resolution)
    # This avoids up-sampling operations that cause multi-view inconsistency
    feature_map = zeros((H, W, C))
    
    for y in range(H):
        for x in range(W):
            ray = rays[y, x]
            F_ray = render_feature_grid_ray(ray.origin, ray.direction, 
                                            scene_model.density_grid, 
                                            scene_model.feature_grid, 
                                            num_samples=scene_model.num_samples)
            feature_map[y, x] = F_ray
            
    # Decode features to RGB using a 2D CNN
    rgb_image = scene_model.cnn_decoder(feature_map)
    
    return rgb_image
```

## Subtopics

### [Shared Feature Grid](17821-shared-feature-grid.md)
A variant that reduces memory usage by repeating a single feature grid across $N$ distinct spatial regions, using region-specific density decoders and a shared color decoder.

## Operating characteristics
*   **Memory Complexity:** The use of vector-matrix tensor decomposition lowers the space complexity of the feature grid from $O(n^3)$ to $O(n^2)$, where $n$ is the grid resolution [S1].
*   **Performance:** Rendering full-resolution feature maps avoids the computational cost and artifacts associated with down-sampling and subsequent up-sampling [S1].
*   **Strengths:**
    *   Eliminates multi-view inconsistency caused by up-sampling operations [S1].
    *   Removes aliasing and severe flickering effects in stylized videos that occur when rendering low-resolution feature maps [S1].
    *   Enables high-level feature manipulation (e.g., style transfer) by decoupling features from RGB [S1].
*   **Limitations:**
    *   Directly using 3D voxel grids is memory-intensive, necessitating the tensor decomposition [S1].
    *   View-dependency effects are disabled to maintain multi-view consistency, which may limit the representation of view-dependent phenomena like specular reflections if not handled by the decoder [S1].

## Background
The Feature Grid 3D Representation was introduced in the context of zero-shot 3D style transfer of Neural Radiance Fields [S1]. It builds upon the original NeRF framework [S1] but modifies the output layer to support high-level feature extraction rather than direct color prediction. The method addresses issues in previous style transfer approaches that used down-sampled feature maps, which led to multi-view inconsistency and flickering [S1].

## Sources
[S1] StyleRF: Zero-Shot 3D Style Transfer of Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00806, fetched 2026-10-26
