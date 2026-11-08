# Epipolar 3D Gaussian Splatting

Also called: Epipolar Transformer-based 3DGS

## Overview
Epipolar 3D Gaussian Splatting is a method for reconstructing 3D Gaussian Splatting (3DGS) scenes from image pairs by incorporating an epipolar transformer to generalize features between two views [S1]. It pairs this feature generalization with a probabilistic prediction of Gaussians aligned along camera-to-pixel rays [S1]. Unlike monocular approaches that estimate depth from a single view, this method leverages stereo geometry to resolve ambiguity, serving as the foundation for multi-view extensions like MVSplat and GS-LRM [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A pair of images (left and right views) and their corresponding camera parameters (intrinsics and extrinsics) [S1].
- **Outputs:** A set of 3D Gaussian parameters (position, covariance, opacity, spherical harmonics coefficients) [S1].

**Data Structures:**
The method utilizes a transformer architecture to process image tokens. The specific internal layout of the transformer is not detailed in the sources, but it processes concatenated tokens from the input images and camera parameters [S1].

**Steps:**
1. **Feature Extraction and Tokenization:** The input images are tokenized. Camera parameters are concatenated with the image tokens [S1].
2. **Epolar Transformer Processing:** The concatenated tokens are passed through a transformer block. The core component is an epipolar transformer designed to generalize features between the two views by leveraging epipolar geometry [S1].
3. **Probabilistic Gaussian Prediction:** The output of the transformer is decoded into Gaussian parameters. This prediction is probabilistic and aligns Gaussians along camera-to-pixel rays [S1].
4. **Rendering and Loss:** The predicted Gaussians are rendered. The reconstruction is supervised by photometric losses such as LPIPS, SSIM, or similar metrics [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def epipolar_3dgs_reconstruction(image_left, image_right, camera_params_left, camera_params_right):
    # Step 1: Tokenization
    # Concatenate image tokens and camera parameters
    tokens_left = tokenize(image_left)
    tokens_right = tokenize(image_right)
    camera_tokens = concatenate(camera_params_left, camera_params_right)
    input_tokens = concatenate(tokens_left, tokens_right, camera_tokens)
    
    # Step 2: Epipolar Transformer
    # Process tokens using an epipolar transformer to generalize features
    transformer_output = epipolar_transformer(input_tokens)
    
    # Step 3: Decode Gaussian Parameters
    # Decode transformer output into Gaussian parameters
    # The prediction is probabilistic and aligned along camera-to-pixel rays
    gaussian_params = decode_gaussians(transformer_output)
    
    # Step 4: Rendering and Loss Calculation
    # Render the scene using the predicted Gaussians
    rendered_image = render(gaussian_params)
    
    # Calculate photometric losses
    loss = calculate_loss(rendered_image, ground_truth_image, metrics=['LPIPS', 'SSIM'])
    
    return gaussian_params, loss
```

**Parameters:**
- **Loss Metrics:** LPIPS, SSIM [S1].
- **Typical Values:** Not given in the sources.

**Edge Cases and Pitfalls:**
- **Camera Pose Accuracy:** The method relies on known camera parameters. Errors in pose estimation may affect the epipolar geometry and feature alignment [S1].
- **Occlusion:** Occluded regions may pose challenges for feature generalization, though the epipolar transformer aims to mitigate this by leveraging stereo cues [S1].

## Subtopics

### [Pose-Aware Projection (PAP)](8527-pose-aware-projection-pap.md)
Pose-Aware Projection (PAP) is a feature aggregation technique used in the fine stage of voxel-selective Gaussian splatting to resolve ambiguous feature assignment in occluded or background regions. Unlike global backprojection which maps 2D features to fixed voxel centers regardless of geometry, PAP projects the positions of selected "skeletal" Gaussians onto 2D feature maps, aligning features with actual 3D surface geometry. This method builds on [Epipolar 3D Gaussian Splatting](6070-epipolar-3d-gaussian-splatting.md) by refining the coarse volumetric features with pose-aligned contextual information, ensuring that features correspond to the same physical location across multiple views.

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Strengths:** Leverages stereo geometry for improved feature generalization compared to monocular methods [S1].
- **Limitations:** Requires known camera poses (unless extended to methods like NoPoSplat) [S1].
- **When to Use:** Suitable for reconstructing 3DGS scenes from image pairs where camera poses are known [S1].
- **When Not to Use:** Not suitable for single-view reconstruction without depth estimation extensions (e.g., Flash3D) [S1].

## Background
Epipolar 3D Gaussian Splatting originates from the integration of epipolar transformers into 3DGS reconstruction pipelines [S1]. It builds upon earlier methods like PixelSplat, which introduced the epipolar transformer for feature generalization between two views [S1]. Subsequent works like MVSplat and GS-LRM extended this approach to multi-view settings and different tokenization strategies [S1]. NoPoSplat later addressed the limitation of requiring known camera poses [S1].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
