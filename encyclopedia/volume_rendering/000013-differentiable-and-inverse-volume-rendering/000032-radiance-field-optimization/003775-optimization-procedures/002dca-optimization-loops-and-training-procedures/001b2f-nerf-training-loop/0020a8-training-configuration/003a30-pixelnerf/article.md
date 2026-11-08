# PixelNeRF

Also called: pixelNeRF, spatial CNN feature

## Overview
PixelNeRF is a radiance-field optimization method that synthesizes NeRF from one or a few reference images by leveraging additional CNN features extracted from those images, rather than relying solely on basic positional encoding [S1, S2]. It outperforms vanilla NeRF in capturing intricate details and semantic consistency, particularly enhancing the reconstruction of internal small structures, though it offers smaller performance margins compared to hash-based methods like Instant-NGP [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** One or more reference images (typically 3 per sampling point in medical applications [S1]), camera poses for these images, and a 3D coordinate $\mathbf{x}$ and viewing direction $\mathbf{d}$ for the target ray.
*   **Outputs:** Volume density $\sigma$ and view-dependent color $\mathbf{c}$ for the sampled points.

**Data Structures:**
*   **CNN Backbone:** A pre-trained ResNet-34 model [S1].
*   **Feature Maps:** Extracted features from the CNN. The feature map size is adjusted from 64 to 128 channels to enhance sensitivity to subtle structures [S1].
*   **MLP:** A Multi-Layer Perceptron with 3 layers and skip connections to the second layer [S1].

**Method Steps:**
1.  **Feature Extraction:** For each reference image, extract low-level features using the CNN backbone [S2].
2.  **Feature Aggregation:**
    *   **Single-View:** The CNN extracts features from the single image. These features are used to predict spatial parameters that define the coordinate system for sampling points [S2].
    *   **Multi-View:** Extract multi-angle projection features from multiple reference views. Aggregate these features using average pooling to form a unified feature representation for the scene [S2].
3.  **Coordinate Encoding:** Combine the aggregated CNN features with the positional encoding of the 3D coordinate $\mathbf{x}$ and viewing direction $\mathbf{d}$ [S1, S2]. The positional encoding uses a higher frequency range (adjusted from 8 to 10) to capture fine medical details [S1].
4.  **MLP Regression:** Pass the concatenated features (CNN features + positional encoding) through the 3-layer MLP with skip connections to predict density $\sigma$ and color $\mathbf{c}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def pixelnerf_render(ray_origin, ray_direction, reference_images, camera_poses):
    """
    Renders a ray using PixelNeRF.
    
    Args:
        ray_origin: 3D vector, origin of the ray.
        ray_direction: 3D vector, direction of the ray.
        reference_images: List of images (H, W, 3), typically 3 images.
        camera_poses: List of 4x4 matrices, camera poses for reference images.
        
    Returns:
        sigma: float, volume density.
        color: 3D vector, view-dependent color.
    """
    # Step 1: Extract CNN features from reference images
    # Using a pre-trained ResNet-34 backbone [S1]
    cnn_features_list = []
    for img in reference_images:
        # Extract features; adjust feature map size from 64 to 128 [S1]
        features = extract_cnn_features(img, backbone='resnet34', feature_size=128)
        cnn_features_list.append(features)
    
    # Step 2: Aggregate features
    # For multi-view, aggregate via average pooling [S2]
    # For single-view, use features to predict spatial parameters [S2]
    if len(reference_images) > 1:
        # Average pooling across views
        aggregated_features = average_pool(cnn_features_list)
    else:
        # Single view: predict spatial parameters from features
        # This defines the coordinate system for sampling points [S2]
        aggregated_features = predict_spatial_params(cnn_features_list[0])
        
    # Step 3: Sample points along the ray
    # Uniform sampling according to volume width [S1]
    sampled_points = sample_points_along_ray(ray_origin, ray_direction, num_samples=256)
    
    # Step 4: Encode coordinates and directions
    # Positional encoding with frequency range 8-10 [S1]
    positional_encoding = positional_encode(sampled_points, freq_min=8, freq_max=10)
    direction_encoding = positional_encode(ray_direction, freq_min=8, freq_max=10)
    
    # Step 5: Combine features and encode
    # Concatenate CNN features with positional encoding [S1, S2]
    combined_input = concatenate(aggregated_features, positional_encoding, direction_encoding)
    
    # Step 6: MLP Regression
    # 3 layers with skip connections to the second layer [S1]
    sigma, color = mlp_predict(combined_input, layers=3, skip_to_layer=2)
    
    return sigma, color

def extract_cnn_features(image, backbone, feature_size):
    # Placeholder for CNN feature extraction logic
    # Uses ResNet-34 pre-trained model [S1]
    # Adjusts feature map size to 'feature_size' [S1]
    pass

def average_pool(features_list):
    # Average pooling across the list of feature maps [S2]
    return sum(features_list) / len(features_list)

def predict_spatial_params(features):
    # Predicts spatial parameters from single view features [S2]
    pass

def positional_encode(coords, freq_min, freq_max):
    # Standard positional encoding with specified frequency range [S1]
    pass

def mlp_predict(input_data, layers, skip_to_layer):
    # 3-layer MLP with skip connections to the second layer [S1]
    # input -> layer1 -> layer2 (skip connection from input) -> layer3 -> output
    pass
```

**Parameters:**
*   **Reference Images:** 3 per sampling point [S1].
*   **CNN Backbone:** ResNet-34 pre-trained [S1].
*   **Feature Map Size:** 128 (adjusted from 64) [S1].
*   **MLP Layers:** 3 layers with skip connections to the second layer [S1].
*   **Positional Encoding Frequency:** 8 to 10 [S1].
*   **Optimizer:** Adam with learning rate $5 \times 10^{-4}$ [S1].
*   **Batch Size:** 1024 rays [S1].
*   **Training Epochs:** 30K, with early termination if evaluation metric deviates by less than 1% for three consecutive instances [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** The reliance on frequency-based encoding means training prioritizes low-frequency components for overall shape, potentially missing high-frequency details if not properly encoded [S1].
*   **Boundaries:** Dense structures (e.g., soft tissues) require more sampling for accurate reconstruction, increasing computational cost [S1].
*   **Degenerate Input:** Single-view synthesis relies on the CNN to predict spatial parameters, which may be less robust than multi-view aggregation [S2].

## Operating characteristics
*   **Time Complexity:** PixelNeRF requires additional computation for CNN feature extraction, leading to higher rendering times compared to vanilla NeRF and Instant-NGP. For example, in the Feet dataset, PixelNeRF took 2.189 s, while Instant-NGP took 1.016 s [S1]. In the Chest dataset, PixelNeRF took 5.493 s, while Instant-NGP took 2.591 s [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** PixelNeRF outperforms vanilla NeRF but has smaller performance margins than Instant-NGP. In the Chest dataset, PixelNeRF achieved a 3D SSIM improvement of 0.031 over vanilla NeRF, compared to Instant-NGP's 0.058 improvement [S1].
*   **Strengths:** Enhances reconstruction of internal small structures (e.g., teeth in the Jaw dataset) [S1]. Improves semantic consistency across diverse views [S1].
*   **Limitations:** Higher computational cost due to CNN feature extraction [S1]. Less effective than hash-based encoding (Instant-NGP) for medical images [S1].
*   **When to Use:** When high-fidelity reconstruction of small internal structures is required and computational resources allow for CNN feature extraction [S1].
*   **When Not to Use:** When real-time rendering or minimal computational cost is prioritized over fine detail reconstruction [S1].

## Background
PixelNeRF was proposed by Yu et al. in 2021 [S2]. It extends the capabilities of vanilla NeRF by leveraging additional CNN features extracted from images [S1]. The method is particularly useful for synthesizing NeRF from one or a few images, addressing the challenge of sparse-view reconstruction [S2]. It has been applied in various domains, including medical image reconstruction, where it helps capture complex spatial relationships within medical images [S1].

## Sources
[S1] The Adaption of Recent New Concepts in Neural Radiance Fields and Their Role for High-Fidelity Volume Reconstruction in Medical Images., https://doi.org/10.3390/s24185923, fetched 2026-10-26
[S2] Overview of NeRF technology and applications, https://www.jcad.cn/en/article/doi/10.3724/SP.J.1089.2023-00759, fetched 2026-10-31
