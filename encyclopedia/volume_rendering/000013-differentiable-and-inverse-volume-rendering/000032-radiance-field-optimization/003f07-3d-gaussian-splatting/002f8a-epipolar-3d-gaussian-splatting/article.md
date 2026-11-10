# Epipolar 3D Gaussian Splatting
## Overview
Epipolar 3D Gaussian Splatting is a method that incorporates an epipolar transformer to generalize features between two views, pairing this with a probabilistic prediction of Gaussians aligned along camera-to-pixel rays [S1]. It solves the problem of generating 3D Gaussian Splatting (3DGS) scenes from limited or specific multi-view inputs by leveraging epipolar geometry to constrain feature aggregation, distinguishing itself from global backprojection methods that ignore actual surface geometry [S1]. It builds on [3D Gaussian Splatting](1795-3d-gaussian-splatting.md) by replacing or augmenting standard densification with transformer-based feature generalization across epipolar lines.

## How it works
**Inputs and Outputs:**
- **Inputs:** Two (or more) input images with known camera poses (intrinsics and extrinsics).
- **Outputs:** A set of 3D Gaussian primitives, each defined by position $\mathbf{x}$, covariance $\Sigma$ (or scale/rotation), opacity $\alpha$, and spherical harmonics coefficients for color/density.

**Data Structures:**
- **Gaussian Primitives:** A list of objects containing:
  - $\mathbf{x} \in \mathbb{R}^3$: Center position.
  - $\mathbf{q} \in \mathbb{R}^4$: Quaternion representing rotation.
  - $\mathbf{s} \in \mathbb{R}^3$: Scale vector.
  - $\alpha \in [0, 1]$: Opacity.
  - $\mathbf{c}$: Color coefficients (e.g., SH coefficients).
- **Feature Maps:** 2D feature tensors extracted from input views, typically via a CNN or Transformer encoder.

**Steps:**
1. **Feature Extraction:** Extract 2D feature maps $F_1, F_2$ from the two input views using a shared encoder network [S1].
2. **Epipolar Transformer:** Apply an epipolar transformer module that aggregates features between the two views. This module projects features from one view onto the epipolar lines of the other view, enforcing geometric consistency [S1]. The transformer outputs generalized feature representations that encode multi-view context.
3. **Probabilistic Gaussian Prediction:** Decode the generalized features into Gaussian parameters. This involves predicting the probability distribution of Gaussians aligned along camera-to-pixel rays. Specifically, for each pixel or ray, the model predicts the likelihood of a Gaussian primitive existing at a certain depth along that ray [S1].
4. **Parameter Generation:** Convert the probabilistic predictions into concrete Gaussian parameters ($\mathbf{x}, \Sigma, \alpha, \mathbf{c}$). The position $\mathbf{x}$ is determined by the depth prediction along the ray. The covariance $\Sigma$ is derived from the scale and rotation parameters, often initialized based on the epipolar geometry or predicted uncertainty.
5. **Rendering:** Render the predicted Gaussians using the standard differentiable rasterization pipeline of 3DGS to produce the output image [S1].
6. **Optimization:** Supervise the rendered images against the input views using photometric losses such as LPIPS, SSIM, or MSE [S1]. Optimize the Gaussian parameters (and potentially the encoder weights if end-to-end) to minimize these losses.

**Formulas:**
- **Epipolar Constraint:** For a point $\mathbf{x}$ in 3D, its projection $x_1$ in view 1 and $x_2$ in view 2 must satisfy $x_2^T F x_1 = 0$, where $F$ is the fundamental matrix. The transformer enforces this by attending to features along the epipolar line $l_2 = F x_1$ [S1].
- **Gaussian Rendering:** The color $C$ of a pixel is computed as:
  $$ C = \sum_{i \in N} c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
  where $N$ is the set of Gaussians sorted by depth, $c_i$ is the color, and $\alpha_i$ is the opacity of the $i$-th Gaussian [S1].

**Parameters:**
- **Transformer Depth/Heads:** Not given in the sources.
- **Loss Weights:** LPIPS, SSIM weights are not specified in the sources.

**Edge Cases and Pitfalls:**
- **Occlusion:** The epipolar transformer may struggle with occluded regions if the feature generalization assumes visibility [S1].
- **Depth Ambiguity:** Probabilistic prediction along rays may produce multiple Gaussians for the same ray; sorting and alpha blending must handle this correctly.
- **Precision:** Floating-point precision in epipolar projection calculations can affect feature alignment.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def epipolar_3dgs(input_img1, input_img2, camera_params):
    # Step 1: Feature Extraction
    F1 = encoder(input_img1)
    F2 = encoder(input_img2)
    
    # Step 2: Epipolar Transformer
    # Aggregate features along epipolar lines
    # Assume get_epipolar_lines computes lines based on camera_params
    epipolar_lines = get_epipolar_lines(camera_params)
    F_generalized = epipolar_transformer(F1, F2, epipolar_lines)
    
    # Step 3: Probabilistic Gaussian Prediction
    # Decode features into Gaussian parameters
    # This step predicts depth probabilities and other attributes
    gaussian_probs = decoder(F_generalized)
    
    # Step 4: Parameter Generation
    # Sample or argmax to get concrete Gaussians
    gaussians = sample_gaussians(gaussian_probs, camera_params)
    
    # Step 5: Rendering
    rendered_img = render_gaussians(gaussians, camera_params)
    
    return rendered_img, gaussians

def get_epipolar_lines(camera_params):
    # Compute fundamental matrix and epipolar lines
    # This is a placeholder for the actual geometric computation
    pass

def sample_gaussians(gaussian_probs, camera_params):
    # Convert probabilistic predictions to Gaussian primitives
    # This involves depth sampling, scale/rotation prediction, etc.
    pass
```

## Subtopics

### [Pose-Aware Projection (PAP)](8527-pose-aware-projection-pap.md)
Pose-Aware Projection (PAP) is a feature aggregation technique used in differentiable volume rendering to extract multi-view contextual information for specific 3D locations (voxels or skeletal Gaussians) by projecting their positions onto 2D feature maps from multiple camera views. It solves the problem of ambiguous feature assignment inherent in global backprojection methods, which ignore actual surface geometry and assume features map reliably to voxel centers via camera rays, leading to errors in occluded or background regions. What sets PAP apart from its parent [Epipolar 3D Gaussian Splatting](6070-epipolar-3d-gaussian-splatting.md) is its focus on aggregating features based on camera distance for *selected* skeletal primitives rather than generalizing features via an epipolar transformer for all pixels. It differs from its sibling [Geometry Aware Feature Extraction](10791-geometry-aware-feature-extraction.md) by using explicit inverse-distance weighting of projected features rather than cross-view attention guided by geometric constraints.

## Operating characteristics
- **Time Complexity:** Depends on the transformer size and number of Gaussians. Transformer attention is $O(N^2)$ or $O(N \log N)$ depending on implementation, where $N$ is the number of pixels/features. Rendering is $O(G)$ where $G$ is the number of Gaussians.
- **Memory Complexity:** Stores feature maps and Gaussian parameters. Memory usage scales with image resolution and number of Gaussians.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of input views and resolution.
- **Strengths:** Leverages epipolar geometry for robust feature generalization; probabilistic prediction allows for flexible Gaussian placement.
- **Limitations:** Requires known camera poses (unless using variants like NoPoSplat); may be computationally intensive due to transformer operations.
- **When to use:** When reconstructing 3DGS from limited multi-view inputs with known poses.
- **When not to use:** When camera poses are unknown (use NoPoSplat) or when real-time performance is critical and transformer overhead is prohibitive.

## Background
Epipolar 3D Gaussian Splatting is part of the broader trend of integrating neural rendering techniques with 3D Gaussian Splatting. It builds upon methods like PixelSplat [S1], which uses an epipolar transformer for feature generalization between two views. MVSplat [S1] extends this to multi-view settings. GS-LRM [S1] tokenizes multiple images and camera parameters for transformer-based decoding. NoPoSplat [S1] addresses the need for known poses. The method is supervised by photometric losses such as LPIPS, SSIM, or similar metrics [S1].

## Sources
[S1] From Volume Rendering to 3D Gaussian Splatting: Theory and Applications, https://doi.org/10.1109/sibgrapi67909.2025.11223276, fetched 2026-10-26
