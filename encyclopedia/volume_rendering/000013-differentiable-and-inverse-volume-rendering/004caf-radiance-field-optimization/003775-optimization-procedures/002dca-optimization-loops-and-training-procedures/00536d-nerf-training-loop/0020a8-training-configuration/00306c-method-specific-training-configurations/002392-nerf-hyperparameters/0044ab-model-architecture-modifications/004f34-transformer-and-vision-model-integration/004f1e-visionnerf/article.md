# VisionNeRF

Also called: Vision Transformer for NeRF

## Overview
VisionNeRF is a novel view synthesis method that infers a 3D radiance field from a single unposed input image by combining global features learned from a Vision Transformer (ViT) and local appearance features extracted from a 2D CNN [S1]. It solves the problem of self-occlusion in image-conditioned models, where standard approaches query features along the ray corresponding to the visible surface in the input view, leading to incorrect conditioning for occluded regions (e.g., using window features to render a wheel) [S1]. Unlike siblings such as [ViT-NeRF](13878-vit-nerf.md) which focuses on multi-view consistency, or [TransNeRF](13907-transnerf.md) which uses transformers for ray sampling, VisionNeRF explicitly replaces the local feature projection paradigm with a hybrid global-local representation to retrieve relevant features from distant parts of the source view for occluded regions [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A single unposed image $I_s$ at camera $s$. No camera pose information is required [S1].
*   **Output:** A rendered novel view image for a target camera position and orientation.

**Data Structures:**
*   **Feature Tokens:** A sequence of latent vectors derived from the ViT encoder.
*   **Multi-level Feature Maps:** A set of 2D tensors at different resolutions (coarse-to-fine) constructed by unflattening and resampling the tokens [S1].
*   **Local Feature Map:** A 2D tensor extracted via a CNN branch [S1].
*   **Hybrid Feature Map:** The fused representation of global and local features [S1].

**Method Steps:**

1.  **Global Feature Extraction:** Lift the input 2D image $I_s$ into feature tokens and apply a Vision Transformer (ViT) to learn global information [S1]. The class token is typically discarded as it lacks spatial meaning for feature maps [S1].
2.  **Feature Map Construction:** Unflatten the feature tokens and resample them into multi-level feature maps to capture global information in a coarse-to-fine manner [S1].
3.  **Local Feature Extraction:** Extract local features from the input image using a 2D convolutional neural network (CNN) to capture details and appearance [S1].
4.  **Feature Fusion:** Fuse the global and local features (e.g., via convolution) into a hybrid feature map [S1].
5.  **Conditioned Rendering:** Train a multi-layer perceptron (MLP) network conditioned on the learned 3D representation (global ViT features and local CNN features) to perform volume rendering for novel viewpoints [S1]. The MLP takes 3D coordinates, viewing directions, and the sampled hybrid features to predict density $\sigma$ and color $\mathbf{c}$ [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def vision_nerf_render(input_image, target_camera_pos, target_camera_dir):
    """
    Renders a novel view from a single unposed input image.
    
    Args:
        input_image: Tensor of shape (H, W, 3) or (3, H, W).
        target_camera_pos: 3D position of the target camera.
        target_camera_dir: Viewing direction vector at the target camera.
        
    Returns:
        rendered_image: Tensor of the novel view.
    """
    
    # Step 1: Global Feature Extraction using ViT
    # Lift image to tokens and pass through ViT layers
    # Discard class token if present
    global_tokens = vit_encoder(input_image)
    
    # Step 2: Feature Map Construction
    # Unflatten tokens and resample into multi-level feature maps
    # This creates a hierarchy (coarse-to-fine)
    global_feature_maps = unflatten_and_resample(global_tokens)
    
    # Step 3: Local Feature Extraction using CNN
    # Extract local appearance details
    local_feature_map = cnn_encoder(input_image)
    
    # Step 4: Feature Fusion
    # Combine global and local features into a hybrid representation
    # Implementation detail: typically involves concatenation or convolution
    hybrid_feature_map = fuse_features(global_feature_maps, local_feature_map)
    
    # Step 5: Conditioned Rendering via NeRF MLP
    # Sample points along rays from target camera
    # Query hybrid features at 3D coordinates
    # Predict density and color
    
    rendered_image = render_volume(hybrid_feature_map, target_camera_pos, target_camera_dir)
    
    return rendered_image

def render_volume(hybrid_feature_map, camera_pos, camera_dir):
    """
    Standard volume rendering loop conditioned on hybrid features.
    """
    # 1. Generate rays from camera_pos and camera_dir
    rays = generate_rays(camera_pos, camera_dir)
    
    # 2. Sample points along each ray
    points = sample_points_along_rays(rays)
    
    # 3. Query hybrid feature map at 3D coordinates
    # Note: Requires a coordinate-to-image-space mapping or implicit feature field
    # Since VisionNeRF uses per-pixel info from feature maps, 
    # the 3D points must be projected or interpolated against the source view's feature map.
    # The source implies an explicit mapping between input image and 3D point queries.
    features = query_hybrid_features(points, hybrid_feature_map)
    
    # 4. MLP predicts density (sigma) and color (c)
    # Inputs: 3D coordinates, viewing direction, sampled features
    density, color = nerf_mlp(points, camera_dir, features)
    
    # 5. Alpha compositing
    image = alpha_composite(rays, density, color)
    
    return image
```

**Parameters:**
*   **Coordinate System:** Viewer-centered coordinate system. The method does not enforce constraints like symmetry or canonical coordinate systems [S1].
*   **Training Data:** ShapeNet dataset (13 object categories) and real-world images [S1].

**Edge Cases and Pitfalls:**
*   **Self-occlusion:** When a target view pixel is invisible from the input view, standard image-conditioned models incorrectly use features from other visible surfaces (e.g., using window features to render a wheel) [S1]. VisionNeRF mitigates this by using self-attention to find long-range dependencies, allowing it to retrieve relevant features from distant parts of the source view for occluded regions [S1].
*   **Blurry Predictions:** Existing approaches using only local image features often render blurry predictions at viewpoints distant from the source view [S1].

## Operating characteristics
*   **Performance:** On the ShapeNet dataset, VisionNeRF achieves LPIPS: 0.09 and PSNR: 21.03 for novel views [S1]. This compares favorably to PixelNeRF, which achieves LPIPS: 0.20 and PSNR: 21.85 [S1].
*   **Strengths:** Renders richer details and more accurate structure in occluded regions compared to local-feature-only models [S1]. Generalizes across multiple object categories using a single model [S1].
*   **Limitations:** Requires training a complex hybrid architecture (ViT + CNN + MLP) [S1].
*   **When to use:** When novel view synthesis from a single unposed image is required, especially for scenes with significant self-occlusion where local feature projection fails [S1].
*   **When not to use:** When camera poses are known and multi-view inputs are available, as other methods may be more efficient or accurate [S1].

## Background
VisionNeRF was proposed to address the limitations of existing image-conditioned models that struggle with self-occlusion and blurry predictions at distant viewpoints [S1]. It builds on the principles of [Transformer and Vision Model Integration](13911-transformer-and-vision-model-integration.md) by leveraging Vision Transformers for global context and CNNs for local details [S1]. The method was trained and evaluated on the ShapeNet dataset, demonstrating state-of-the-art performance against approaches like SRN, PixelNeRF, FE-NVS, SRT, and FWD [S1]. Unlike SRT, which learns 3D scene information as a set of latent code decoded by a transformer, VisionNeRF adopts a radiance field representation using per-pixel information from feature maps, providing an explicit mapping between the input image and 3D point queries [S1].

## Sources
[S1] Vision Transformer for NeRF-Based View Synthesis from a Single Input Image, https://doi.org/10.1109/wacv56688.2023.00087, fetched 2026-10-27
