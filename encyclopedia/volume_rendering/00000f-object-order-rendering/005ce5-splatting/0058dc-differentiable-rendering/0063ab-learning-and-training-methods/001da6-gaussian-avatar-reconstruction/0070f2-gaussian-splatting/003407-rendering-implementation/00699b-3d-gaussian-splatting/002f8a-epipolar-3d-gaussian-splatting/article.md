# Epipolar 3D Gaussian Splatting
## Overview
Epipolar 3D Gaussian Splatting is a method that incorporates an epipolar transformer to generalize features between two views, pairing this with a probabilistic prediction of Gaussians aligned along camera-to-pixel rays [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** Two or more input images and their camera parameters (intrinsics and extrinsics) [S1].
- **Outputs:** A set of 3D Gaussian primitives (position, scale, rotation, opacity, and color coefficients) [S1, G].

**Data Structures:**
- **Gaussian Primitives:** A list of objects containing position $\mathbf{x} \in \mathbb{R}^3$, scale $\mathbf{s} \in \mathbb{R}^3$, rotation $\mathbf{q} \in \mathbb{R}^4$, opacity $\alpha \in [0, 1]$, and color coefficients $\mathbf{c}$ [G].
- **Feature Tokens:** Concatenated sequence of image and camera parameter embeddings processed by the transformer [S1].

**Steps:**
1. **Tokenization:** Tokenize the input images and camera parameters, then concatenate the tokens from all views [S1].
2. **Feature Generalization:** Pass the concatenated tokens through a transformer block to generalize features between the views [S1].
3. **Probabilistic Prediction:** Decode the transformer output tokens into Gaussian parameters, predicting a probabilistic distribution of Gaussians aligned along camera-to-pixel rays [S1].
4. **Rendering:** Render the predicted Gaussians using differentiable rasterization to produce the output image [S1].
5. **Optimization:** Optimize the Gaussian parameters (and encoder weights if trained end-to-end) by minimizing photometric losses such as LPIPS or SSIM against the input views [S1].

**Formulas:**
Not given in the sources.

**Parameters:**
Specific hyperparameters such as transformer depth, heads, and loss weights are not detailed in the source.

**Edge Cases and Pitfalls:**
- **Pose Requirement:** Standard implementation requires known camera poses, though variants like NoPoSplat eliminate this need [S1].
- **Depth Ambiguity:** Probabilistic prediction along rays may produce multiple Gaussians per ray; standard sorting and alpha blending must handle this correctly [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def epipolar_3dgs(images, camera_intrinsics, camera_extrinsics):
    # 1. Tokenization: Embed image patches and concatenate with camera pose embeddings
    tokens = []
    for img, intr, extr in zip(images, camera_intrinsics, camera_extrinsics):
        # [G] Standard vision transformer patch embedding and positional encoding
        tokens.append(embed_and_concat(img, intr, extr))
    combined_tokens = torch.cat(tokens, dim=1)

    # 2. Feature Generalization: Process tokens through a transformer block
    # [S1] Epipolar transformer generalizes features between views
    # [S1] Exact architectural details (layers, heads, dimensions) are not specified
    transformer_output = transformer_forward(combined_tokens)

    # 3. Probabilistic Prediction: Decode tokens into Gaussian parameters
    # [S1] Predicts Gaussians aligned along camera-to-pixel rays
    # [G] MLP decoder outputs position, scale, rotation, opacity, color
    gaussians = decode_to_gaussians(transformer_output)

    # 4. Rendering: Differentiable rasterization
    # [G] Project 3D Gaussians to 2D, sort by depth, alpha blend
    rendered_image = rasterize_gaussians(gaussians, camera_intrinsics, camera_extrinsics)

    return rendered_image, gaussians
```

## Subtopics

### [Pose-Aware Projection (PAP)](8527-pose-aware-projection-pap.md)
Pose-Aware Projection (PAP) is a feature aggregation technique used in differentiable volume rendering to extract multi-view contextual information for specific 3D locations (voxels or skeletal Gaussians) by projecting their positions onto 2D feature maps from multiple camera views. It solves the problem of ambiguous feature assignment inherent in global backprojection methods, which ignore actual surface geometry and assume features map reliably to voxel centers via camera rays, leading to errors in occluded or background regions. What sets PAP apart from its parent [Epipolar 3D Gaussian Splatting](6070-epipolar-3d-gaussian-splatting.md) is its focus on aggregating features based on camera distance for *selected* skeletal primitives rather than generalizing features via an epipolar transformer for all pixels. It differs from its sibling [Geometry Aware Feature Extraction](10791-geometry-aware-feature-extraction.md) by using explicit inverse-distance weighting of projected features rather than cross-view attention guided by geometric constraints.

## Operating characteristics
- **Scaling:** Scales with the number of input views and image resolution [S1].
- **Strengths:** Leverages epipolar geometry for robust feature generalization; probabilistic prediction allows for flexible Gaussian placement [S1].
- **Limitations:** Requires known camera poses in standard formulations, though pose-agnostic variants exist [S1].
- **When to use:** When reconstructing 3DGS from limited multi-view inputs with known poses [S1].
- **When not to use:** When camera poses are unknown (use NoPoSplat) or when transformer overhead is prohibitive [S1].

## Background
Epipolar 3D Gaussian Splatting is part of the broader trend of integrating neural rendering techniques with 3D Gaussian Splatting [S1]. It builds upon methods like PixelSplat, which uses an epipolar transformer for feature generalization between two views [S1]. MVSplat extends this to multi-view settings using transformers and cost volumes [S1]. GS-LRM tokenizes multiple images and camera parameters for transformer-based decoding [S1]. NoPoSplat addresses critical limitations by eliminating the need for known camera poses [S1]. The method is supervised by photometric losses such as LPIPS, SSIM, or similar metrics [S1].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
