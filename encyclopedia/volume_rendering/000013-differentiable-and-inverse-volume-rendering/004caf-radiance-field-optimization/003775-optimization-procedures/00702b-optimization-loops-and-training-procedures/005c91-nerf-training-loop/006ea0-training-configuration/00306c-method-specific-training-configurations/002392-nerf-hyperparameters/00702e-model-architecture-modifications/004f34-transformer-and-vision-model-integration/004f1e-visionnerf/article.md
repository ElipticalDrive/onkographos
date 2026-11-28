# VisionNeRF

Also called: Vision Transformer for NeRF

## Overview
VisionNeRF is a novel view synthesis method that infers a 3D representation from a single unposed input image by combining global features learned from a Vision Transformer (ViT) and local appearance features extracted from a 2D CNN, rendered via a NeRF-based MLP [S1]. Unlike sibling methods such as [ViT-NeRF](13878-vit-nerf.md) which may focus on different architectural integrations, VisionNeRF specifically addresses the degradation of rendering quality in occluded regions by using self-attention to retrieve relevant features from distant parts of the source view, rather than querying features along the ray corresponding to the visible surface [S1]. It differs from [TransNeRF](13907-transnerf.md) by adopting a radiance field representation with per-pixel information from feature maps, providing an explicit mapping between the input image and 3D point queries, whereas SRT learns 3D scene information as a set of latent codes decoded by a transformer [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single unposed image $I_s$ at camera $s$. No camera pose information is required [S1].
*   **Output:** A rendered image for a target viewpoint defined by a ray origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1].

**Data Structures:**
1.  **Feature Tokens:** A sequence of vectors obtained by lifting the input image into patches and processing them through the ViT.
2.  **Multi-level Feature Maps:** 2D tensor grids at multiple resolutions, derived from unflattened and resampled ViT tokens, capturing global information in a coarse-to-fine manner [S1].
3.  **Local Feature Maps:** 2D tensor grids extracted from a 2D CNN, capturing local details and appearance [S1].

**Numbered Steps:**

1.  **Global Feature Extraction:** Lift the input 2D image $I_s$ into feature tokens and apply a Vision Transformer (ViT) to learn global information [S1].
2.  **Feature Map Resampling:** Unflatten the ViT feature tokens and resample them into multi-level feature maps to capture global information in a coarse-to-fine manner [S1].
3.  **Local Feature Extraction:** Extract local features from the input image $I_s$ using a 2D convolutional neural network (CNN) to capture details and appearance [S1].
4.  **3D Representation Conditioning:** For a query 3D point $\mathbf{x}$ and viewing direction $\mathbf{d}$, condition a Multi-Layer Perceptron (MLP) on the learned 3D representation. This representation is formed by combining the global features (from the multi-level feature maps) and local features (from the CNN) [S1].
5.  **Volume Rendering:** Use the MLP to predict volume density $\sigma$ and view-dependent color $\mathbf{c}$ for the query point, then integrate these values along the ray using standard volume rendering equations [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vision_nerf_render(input_image, ray_origin, ray_direction):
    """
    Renders a novel view from a single unposed image using VisionNeRF.
    
    Args:
        input_image: 2D tensor of shape (H, W, 3) representing the single input image I_s.
        ray_origin: 3D vector o for the target viewpoint ray.
        ray_direction: 3D vector d for the target viewpoint ray.
        
    Returns:
        color: Rendered color value for the ray.
        depth: Integrated depth value for the ray.
    """
    
    # Step 1: Global Feature Extraction via Vision Transformer
    # Lift image to tokens and apply ViT
    # Note: Specific ViT architecture (e.g., patch size, number of layers) 
    # is not detailed in the source notes. [G]
    tokens = vision_transformer_forward(input_image)
    
    # Step 2: Resample to Multi-level Feature Maps
    # Unflatten tokens and resample into coarse-to-fine feature maps
    # Note: The specific resampling mechanism (e.g., bilinear, nearest) 
    # and number of levels are not detailed in the source notes. [G]
    global_feature_maps = unflatten_and_resample(tokens)
    
    # Step 3: Local Feature Extraction via CNN
    # Extract local features capturing details and appearance
    # Note: Specific CNN architecture (e.g., ResNet, EfficientNet) 
    # is not detailed in the source notes. [G]
    local_feature_maps = cnn_forward(input_image)
    
    # Step 4: Sample Features for Query Point
    # Query the global and local feature maps at the 3D point's projection
    # Note: The method uses an explicit mapping between input image and 3D point.
    # Since no camera pose is provided, the projection of x onto the input image 
    # plane must be inferred or handled implicitly by the conditioning network.
    # The source states it does not enforce symmetry or canonical coordinate systems.
    # [G]
    global_features = sample_features(global_feature_maps, ray_origin, ray_direction)
    local_features = sample_features(local_feature_maps, ray_origin, ray_direction)
    
    # Step 5: MLP Conditioning and Volume Rendering
    # Condition MLP on global and local features
    # The MLP takes (global_features, local_features, ray_direction) as input
    # to predict density and color.
    # Note: The exact MLP input concatenation and structure are not detailed. [G]
    density, color = mlp_forward(global_features, local_features, ray_direction)
    
    # Standard Volume Rendering Integration along the ray
    # Note: Standard NeRF integration is assumed.
    color, depth = volume_render(ray_origin, ray_direction, density, color)
    
    return color, depth
```

**Parameters and Typical Values:**
*   **Dataset:** ShapeNet (13 object categories) and real-world images [S1].
*   **Performance (ShapeNet):** LPIPS: 0.09, PSNR: 21.03 [S1].
*   **Comparison Baseline (PixelNeRF on ShapeNet):** LPIPS: 0.20, PSNR: 21.85 [S1].

**Edge Cases and Pitfalls:**
*   **Self-occlusion:** In standard image-conditioned models, when a target view pixel is invisible from the input view, the model incorrectly uses features from other visible surfaces (e.g., using window features to render a wheel) [S1]. VisionNeRF mitigates this by using self-attention to find long-range dependencies, allowing it to retrieve relevant features from distant parts of the source view for occluded regions [S1].
*   **Coordinate System:** The method uses a viewer-centered coordinate system and does not enforce constraints like symmetry or canonical coordinate systems [S1].

## Operating characteristics
*   **Strengths:** Renders richer details and more accurate structure in occluded regions compared to existing approaches like PixelNeRF [S1]. Generalizes across multiple object categories using a single model [S1].
*   **Limitations:** Performance on PSNR is slightly lower than PixelNeRF on ShapeNet (21.03 vs 21.85), though LPIPS is significantly better (0.09 vs 0.20), indicating superior perceptual quality [S1].
*   **When to use:** When novel view synthesis from a single unposed image is required, particularly when occluded regions need to be rendered with high fidelity.
*   **When not to use:** When camera pose information is available and precise PSNR is prioritized over perceptual quality in non-occluded regions, as PixelNeRF may offer higher PSNR [S1].

## Background
VisionNeRF was introduced to address the problem of novel view synthesis from a single unposed image, where existing approaches using local image features often render blurry predictions at viewpoints distant from the source view due to self-occlusion [S1]. The method leverages advances in Vision Transformers (ViT) and Neural Radiance Fields (NeRF) to learn a better 3D representation by integrating global and local information [S1]. It was trained and evaluated on the ShapeNet dataset including 13 object categories, as well as real-world images [S1]. The work was done while the authors were interning at Google or at Google [S1].

## Sources
[S1] Vision Transformer for NeRF-Based View Synthesis from a Single Input Image, https://doi.org/10.1109/wacv56688.2023.00087, fetched 2026-10-27
