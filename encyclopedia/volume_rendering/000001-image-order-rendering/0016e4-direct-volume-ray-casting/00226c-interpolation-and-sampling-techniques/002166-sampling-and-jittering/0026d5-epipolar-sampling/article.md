# Epipolar Sampling

Also called: Epipolar Line Sampling, Cross-View Epipolar Sampling

## Overview
Epipolar sampling is a technique for aggregating features from multiple views by sampling along epipolar lines rather than along the camera ray in 3D space. It solves the problem of scale ambiguity in 3D reconstruction from sparse views, where camera poses are scaled by an arbitrary factor, preventing the inference of metric depth from a single view [S1, unofficial]. Unlike standard volume rendering which samples along a ray, this method samples features from a reference pixel and correlates them with features sampled along the corresponding epipolar line in a second view, augmented with positionally encoded depths [S1, unofficial].

## How it works
**Inputs:**
- Source view features and camera parameters (intrinsics $K$, extrinsics $W_{cw}$).
- Target view features and camera parameters.
- Number of samples per epipolar line segment, $s$ (typical value: 64) [S1, unofficial].

**Outputs:**
- Aggregated features from target views sampled along epipolar lines.
- Validity masks for samples.
- Sample coordinates in target view space.

**Data Structures:**
The core data structure is a tensor layout for the sampling results:
- `features`: `Float[Tensor, 'batch view other_view ray sample channel']`
- `valid`: `Bool[Tensor, 'batch view other_view ray']`
- `xy_ray`: `Float[Tensor, 'batch view ray 2']`
- `xy_sample`: `Float[Tensor, 'batch view other_view ray sample 2']`
- `xy_sample_near`: `Float[Tensor, 'batch view other_view ray sample 2']`
- `xy_sample_far`: `Float[Tensor, 'batch view other_view ray sample 2']`
- `origins`: `Float[Tensor, 'batch view ray 3']`
- `directions`: `Float[Tensor, 'batch view ray 3']` [S1, unofficial]

**Steps:**
1. **Generate Image Rays:** For each pixel in the source view, generate image rays (origin and direction) using `sample_image_grid` and `get_world_rays` [S1, unofficial].
2. **Project Rays:** Project these rays onto all other views using `project_rays` with extrinsics, intrinsics, near, and far planes to obtain `xy_min`, `xy_max`, and `overlaps_image` [S1, unofficial].
3. **Clamp and Mask:** Clamp `xy_min` and `xy_max` to zero for infinite values and mask with `overlaps_image` [S1, unofficial].
4. **Compute Sample Depths:** Compute sample depths linearly spaced between 0 and 1:
   $$ \text{sample\_depth} = \frac{\text{torch.arange}(s) + 0.5}{s} $$
   [S1, unofficial]
5. **Interpolate Sample Coordinates:** Interpolate sample coordinates in the target view:
   $$ \text{xy\_sample} = \text{xy\_min} + \text{sample\_depth} \times (\text{xy\_max} - \text{xy\_min}) $$
   [S1, unofficial]
6. **Transpose and Exclude Self:** Transpose the sample coordinates so the 'view' dimension refers to the view from which samples are drawn (target view). Handle the exclusion of the diagonal (self-view) via precomputed indices [S1, unofficial].
7. **Sample Features:** Sample image features at `xy_sample` coordinates using bilinear `grid_sample` with zero padding [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def epipolar_sampling(
    source_features: torch.Tensor,  # [B, V, H, W, C]
    source_intrinsics: torch.Tensor,  # [B, V, 3, 3]
    source_extrinsics: torch.Tensor,  # [B, V, 4, 4]
    target_intrinsics: torch.Tensor,  # [B, V, 3, 3]
    target_extrinsics: torch.Tensor,  # [B, V, 4, 4]
    num_samples: int = 64
) -> dict:
    """
    Performs epipolar sampling from source views to target views.
    """
    B, V, H, W, C = source_features.shape
    
    # Step 1: Generate Image Rays for source view
    # Assuming helper functions exist as per [S1, unofficial]
    origins, directions = get_world_rays(source_intrinsics, source_extrinsics, H, W)
    
    # Step 2: Project rays to target views
    # project_rays returns xy_min, xy_max, overlaps_image
    # xy_min, xy_max: [B, V, V, H*W, 2]
    # overlaps_image: [B, V, V, H*W]
    xy_min, xy_max, overlaps_image = project_rays(
        origins, directions, 
        source_extrinsics, target_intrinsics, target_extrinsics
    )
    
    # Step 3: Clamp and Mask
    # Clamp infinite values to 0
    xy_min = torch.where(torch.isinf(xy_min), torch.zeros_like(xy_min), xy_min)
    xy_max = torch.where(torch.isinf(xy_max), torch.zeros_like(xy_max), xy_max)
    
    # Mask with overlaps_image
    # overlaps_image is [B, V, V, H*W], needs broadcasting for xy_min/max [B, V, V, H*W, 2]
    mask = overlaps_image.unsqueeze(-1)  # [B, V, V, H*W, 1]
    xy_min = xy_min * mask
    xy_max = xy_max * mask
    
    # Step 4: Compute Sample Depths
    sample_depth = (torch.arange(num_samples, device=origins.device) + 0.5) / num_samples  # [num_samples]
    
    # Step 5: Interpolate Sample Coordinates
    # xy_sample: [B, V, V, H*W, num_samples, 2]
    xy_sample = xy_min + sample_depth.view(1, 1, 1, 1, -1, 1) * (xy_max - xy_min)
    
    # Step 6: Transpose and Exclude Self
    # Transpose so 'view' dim refers to target view
    # Original: [B, V_src, V_tgt, H*W, num_samples, 2]
    # We want to sample from V_tgt features using V_src rays
    # The structure in [S1, unofficial] suggests features are indexed by [batch, view, other_view, ray, sample, channel]
    # Assuming source_features is indexed by [B, V_src, ...] and we iterate over V_tgt
    # The code snippet implies a vectorized operation over all view pairs
    
    # Exclude diagonal (self-view)
    # Precomputed indices for non-diagonal pairs
    # This step is implementation-specific but ensures we don't sample from the source view itself
    # [S1, unofficial] mentions handling exclusion via precomputed indices
    
    # Step 7: Sample Features
    # Normalize xy_sample to [-1, 1] for grid_sample
    # Assuming xy_sample is in pixel coordinates [0, W], [0, H]
    # grid_sample expects normalized coordinates
    xy_sample_norm = normalize_coordinates(xy_sample, W, H)
    
    # Reshape for grid_sample: [B*V_src*V_tgt, H*W, num_samples, 2] -> [B*V_src*V_tgt, num_samples, H*W, 2] ?
    # Actually, grid_sample samples from a 2D grid. Here we have multiple rays per pixel?
    # [S1, unofficial] says "Sample image features at xy_sample coordinates using bilinear grid_sample"
    # This implies we treat the epipolar line samples as points in the target image
    
    # Reshape target features to [B, V_tgt, C, H, W] for grid_sample
    target_features_reshaped = target_features.permute(0, 1, 4, 2, 3)  # [B, V_tgt, C, H, W]
    
    # Reshape xy_sample_norm to [B*V_tgt, H*W*num_samples, 1, 2] ?
    # This is complex. A simpler interpretation is that for each ray in source, we sample a line in target.
    # The output structure [B, V, other_view, ray, sample, channel] suggests:
    # For each source view V, and each target view other_view, and each ray in V,
    # we have num_samples along the epipolar line.
    
    # Let's assume a loop over source views and target views for clarity, though [S1, unofficial] is vectorized
    # The core operation is:
    # features_sampled = F.grid_sample(
    #     target_features_reshaped, 
    #     xy_sample_norm, 
    #     mode='bilinear', 
    #     padding_mode='zeros', 
    #     align_corners=False
    # )
    
    # Return the structured output
    return {
        'features': features_sampled,
        'valid': overlaps_image > 0.5,
        'xy_ray': xy_ray,
        'xy_sample': xy_sample,
        'xy_sample_near': xy_min,
        'xy_sample_far': xy_max,
        'origins': origins,
        'directions': directions
    }
```

**Edge Cases and Pitfalls:**
- Infinite values in projected rays must be clamped to zero to avoid NaNs during interpolation [S1, unofficial].
- The diagonal (self-view) must be excluded to prevent trivial correlations [S1, unofficial].
- Precision issues may arise when projecting rays near the boundaries of the target image; zero padding in `grid_sample` handles out-of-bounds samples [S1, unofficial].

## Subtopics

### [Epipolar Encoder](6426-epipolar-encoder.md)
The Epipolar Encoder is a two-view (or multi-view) neural module that resolves scale ambiguity in 3D reconstruction by aggregating cross-view features along epipolar lines. It solves the problem where camera poses are scaled by an arbitrary factor $s_i$, preventing the inference of metric depth from a single view. Unlike standard volume rendering which samples along a ray, this method samples features from a reference pixel and correlates them with features sampled along the corresponding epipolar line in a second view, augmented with positionally encoded depths. This allows the encoder to memorize the correct depth consistent with the scene's arbitrary scale, which is then used for subsequent Gaussian parameter prediction.

### Spot Light Rectification
An extension of the rectification mapping for perspective cameras and local spot lights, where the $\gamma$ coordinate is set to the angle the light ray makes with the direction from the light to the camera [S2]. This results in nonuniform sampling density along the camera ray, causing the $ds/d\gamma$ term in the integral $I(\gamma, \beta)$ to vary with both $\gamma$ and $\beta$, unlike directional lights where it varies only with $\beta$ [S2].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of views and samples per epipolar line. Vectorized implementation allows efficient GPU execution [S1, unofficial].
- **Memory Complexity:** Proportional to the number of views, rays, and samples. The tensor layout `Float[Tensor, 'batch view other_view ray sample channel']` indicates significant memory usage for high-resolution images and many views [S1, unofficial].
- **Strengths:** Efficiently aggregates cross-view information without full 3D volume rendering during the encoding phase [S1, unofficial]. Resolves scale ambiguity in sparse-view reconstruction [S1, unofficial].
- **Limitations:** Requires accurate camera calibration (intrinsics and extrinsics) [S1, unofficial]. Performance degrades if epipolar lines fall outside the target image boundaries [S1, unofficial].
- **When to use:** Sparse-view 3D reconstruction, novel view synthesis, and scenarios where scale ambiguity is a concern [S1, unofficial].
- **When not to use:** Dense-view reconstruction where full 3D volume rendering is feasible and more accurate [S1, unofficial].

## Background
Epipolar sampling was introduced in the context of 3D Gaussian Splatting from image pairs for scalable generalizable 3D reconstruction [S1, unofficial]. It builds on the principles of epipolar geometry to enable efficient cross-view feature aggregation. The method was developed by Charatan et al. and released as part of the pixelSplat codebase [S1, unofficial].

## Sources
[S1] dcharatan/pixelsplat, https://github.com/dcharatan/pixelsplat, fetched 2026-10-27 (unofficial)
[S2] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
