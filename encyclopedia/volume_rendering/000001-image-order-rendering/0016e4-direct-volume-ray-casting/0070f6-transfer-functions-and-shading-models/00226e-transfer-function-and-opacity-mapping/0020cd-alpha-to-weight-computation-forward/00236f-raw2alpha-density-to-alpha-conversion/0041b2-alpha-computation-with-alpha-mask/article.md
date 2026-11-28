# Alpha Computation with Alpha Mask

Also called: Masked Polygons and $\bar{\alpha}$, $\bar{\alpha}$ Occlusion Weighing

## Overview
This method computes alpha values for volume rendering or occlusion estimation by gating density or opacity accumulation with a binary mask derived from an alpha texture [S2, unofficial]. It solves the problem of over-estimating occlusion for planar surfaces with complex contours (e.g., foliage, fences) where the underlying geometry does not match the actual occlusion properties [S3, unofficial]. Unlike methods that apply uniform opacity, this approach weighs occlusion by an average alpha value $\bar{\alpha}$ fetched efficiently from a low MIP level texture [S3, unofficial].

## How it works
**Inputs:**
- `xyz_locs`: 3D coordinates tensor of shape $[N, 3]$ [S2, unofficial].
- `alpha_mask_texture`: Alpha mask texture or tensor associated with the surface [S2, unofficial].
- `features`: Density features or raw density values at `xyz_locs` [S2, unofficial].
- `length`: Integration length, scalar or tensor of shape $[N]$ [S2, unofficial].
- `component_wise_feature2density`: Boolean flag indicating if features are already densities [S2, unofficial].

**Outputs:**
- `sigma`: Absorption coefficient tensor of shape $[N]$ with zeros in masked-out regions [S2, unofficial].
- `alpha`: Alpha tensor of shape $[N]$ computed from valid sigma values [S2, unofficial].

**Data Structures:**
- `alpha_mask`: Boolean tensor of shape $[N]$, `True` where location is within valid alpha region [S2, unofficial].
- `sigma`: Float tensor of shape $[N]$, initialized to zeros [S2, unofficial].

**Steps:**
1. **Mask Generation:** Sample alpha values at `xyz_locs` from `alpha_mask_texture`. Create boolean `alpha_mask` where sampled alpha > 0 [S2, unofficial].
2. **Initialization:** Initialize `sigma` tensor with zeros [S2, unofficial].
3. **Valid Point Check:** Check if `alpha_mask.any()`. If false, return zero alpha [S2, unofficial].
4. **Feature Processing:** For valid points, normalize coordinates and compute density features. If `component_wise_feature2density` is true, use features directly as `sigma_feature`; otherwise, apply `feature2density` conversion [S2, unofficial].
5. **Sigma Assignment:** Assign `sigma_feature` values to `sigma` at indices where `alpha_mask` is true [S2, unofficial].
6. **Alpha Computation:** Compute alpha using the Beer-Lambert law [S2, unofficial].

**Formula:**
$$ \alpha = 1 - \exp(-\sigma \cdot \text{length}) $$
where $\alpha$ is the alpha value, $\sigma$ is the absorption coefficient, and $\text{length}$ is the integration length [S2, unofficial].

**Parameters:**
- `component_wise_feature2density`: Boolean. If `True`, uses features directly as density; otherwise applies conversion [S2, unofficial].

**Edge Cases and Pitfalls:**
- If `alpha_mask` is all false, the result is all zeros [S2, unofficial].
- Precision issues may arise if $\sigma \cdot \text{length}$ is very large, causing $\exp$ to underflow to zero, resulting in $\alpha=1$ [G].
- Cannot efficiently use the full alpha-channel for positional variation without tessellation, because the analytic solution assumes constant opacity over the integration domain [S3, unofficial].
- The choice of classification algorithm impacts result quality and computational time; grayscale data assigns pixels faster than RGBA data [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_alpha_with_mask(xyz_locs, alpha_mask_texture, features, length, component_wise_feature2density, feature2density_func):
    # Step 1: Generate mask
    # [G] Standard texture sampling returns alpha values for each coordinate
    alpha_values = fetch_texture_values(alpha_mask_texture, xyz_locs)
    alpha_mask = [v > 0.0 for v in alpha_values]

    # Step 2: Initialize sigma tensor
    sigma = [0.0] * len(xyz_locs)

    # Step 3: Valid point check
    if not any(alpha_mask):
        return [0.0] * len(xyz_locs)

    # Step 4 & 5: Feature processing and sigma assignment
    valid_indices = [i for i, m in enumerate(alpha_mask) if m]
    valid_features = [features[i] for i in valid_indices]

    if component_wise_feature2density:
        valid_sigma = [f[0] for f in valid_features]
    else:
        valid_sigma = feature2density_func(valid_features)

    for i, idx in enumerate(valid_indices):
        sigma[idx] = valid_sigma[i]

    # Step 6: Alpha computation
    alpha = [1.0 - math.exp(-s * l) for s, l in zip(sigma, length)]

    return alpha

def fetch_texture_values(texture, coords):
    # [G] Standard texture sampling implementation
    return [0.0] * len(coords)
```

## Subtopics

### [Alpha Discard](14786-alpha-discard.md)
Alpha Discard is a fragment-level optimization that discards fragments with alpha values below a specified threshold to handle transparency. It enables the renderer to skip blending operations for nearly transparent pixels, allowing the rendering engine to pass through to opaque geometry behind them, a technique known as alpha clip. It is distinct from Alpha Blend, which computes actual transparency but suffers from sorting and performance limitations with multiple surfaces. Choose it when rendering performance is critical and near-transparent pixels can be safely discarded; avoid it when accurate transparency accumulation is required.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of sample points, dominated by texture sampling and the exponential function [S2, unofficial].
- **Memory Complexity:** $O(N)$ for storing the `sigma` and `alpha` tensors [S2, unofficial].
- **Strengths:** Efficiently handles complex geometries like foliage by preventing over-estimation of occlusion [S3, unofficial].
- **Limitations:** Requires a pre-computed or accessible alpha mask. Cannot efficiently handle positional variation within a single polygon without tessellation, as the analytic solution assumes constant opacity over the integration domain [S3, unofficial].
- **When to use:** When rendering masked polygons (e.g., fences, foliage) where the geometry does not match the occlusion properties [S3, unofficial].
- **When not to use:** When high-frequency positional variation in opacity is required within a single polygon without tessellation [S3, unofficial].

## Background
The concept of using an average alpha value $\bar{\alpha}$ to weigh occlusion for masked polygons was introduced in the context of Ambient Occlusion Volumes (AOV) to correct over-estimation of occlusion for surfaces like foliage and fences [S3, unofficial]. The implementation details involving tensor-based computation and feature conversion appear in unofficial repositories such as Joint-TensoRF [S2, unofficial]. Classification-based alpha generation for depth perception in virtual microscopy was also explored using k-means clustering and back-to-front blending [S1].

## Sources
[S1] Simulating Depth Perception in Virtual Microscopy, https://doi.org/10.1109/ISBI.2019.8759160, fetched 2026-10-26
[S2] Nemo1999/Joint-TensoRF, https://github.com/Nemo1999/Joint-TensoRF, fetched 2026-10-27 (unofficial)
[S3] Ambient Occlusion Volumes, https://research.nvidia.com/sites/default/files/pubs/2010-06_Ambient-Occlusion-Volumes/McGuire10AOV.pdf, fetched 2026-10-28 (unofficial)
