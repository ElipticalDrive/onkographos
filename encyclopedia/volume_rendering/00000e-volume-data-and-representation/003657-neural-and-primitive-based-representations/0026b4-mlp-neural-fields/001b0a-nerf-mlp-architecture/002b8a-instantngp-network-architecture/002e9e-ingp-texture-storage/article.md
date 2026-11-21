# iNGP Texture Storage
Also called: Texture Storage for Real-Time Rendering, iNGP Texture Storage for Real-Time Rendering
## Overview
This method replaces hash-table-based feature storage with explicit 3D grid and triplane textures to leverage hardware-accelerated memory locality and bilinear interpolation [S1]. It reduces the number of texture fetches per neural network evaluation by storing presummed multi-resolution features, enabling a 3× reduction in fetch overhead compared to the original InstantNGP approach [S1].
## How it works
**Inputs and Outputs:**
- Input: A set of multi-resolution feature grids (3D) and triplanes (2D) optimized during training.
- Output: Anti-aliased feature vectors for a given 3D sample point $x$ at render time.

**Data Structures:**
- **Feature Grids/Triplanes:** Explicit 3D or 2D arrays of floating-point values. Unlike hash tables, these are stored as textures in GPU memory.
- **Presummed Features:** For each resolution level $L$, the stored value $g'(v)$ is the sum of features from level $0$ to $L$:
  $$g'(v) = \sum_{l=0}^{L} g(v, l)$$
  where $g(v, l)$ is the feature at location $v$ and resolution level $l$ [S1].

**Algorithm:**
1. **Training Phase:** Optimize the multi-resolution feature grids and triplanes. During this phase, features at different resolution levels are summed to form the presummed features $g'$ stored in the texture memory [S1].
2. **Render-Time Fetch:** For a sample point $x$, determine the required resolution based on the pixel area $p(x)$ [S1].
3. **Interpolation:** Perform bilinear (or trilinear) interpolation on the presummed feature textures at the two resolution levels implied by $p(x)$ [S1]. This interpolation effectively averages the features across resolutions, providing anti-aliasing [S1].
4. **MLP Input:** Pass the interpolated feature vector to the MLP for color/density prediction [S1].

**Parameters:**
- **Resolution Levels:** The number of levels depends on the scene complexity and desired quality. In the cited example, seven triplane levels and three 3D grids were used [S1].
- **Pixel Area Threshold:** Determines which two resolution levels to interpolate between based on $p(x)$ [S1].

**Edge Cases and Pitfalls:**
- **Memory Locality:** While this method improves locality over hash tables, it still requires careful management of texture memory size for high-resolution grids [S1].
- **Interpolation Artifacts:** Bilinear interpolation may introduce slight blurring compared to nearest-neighbor hash lookups, but this is mitigated by the multi-resolution presummed storage [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_feature_anti_aliased(x, pixel_area_p, feature_grids, feature_triplanes):
    """
    Retrieve anti-aliased features for point x using presummed textures.
    
    Args:
        x: 3D point coordinates.
        pixel_area_p: Area of the pixel in 3D space, determines resolution.
        feature_grids: List of 3D feature grids, stored as presummed textures.
        feature_triplanes: List of 2D triplane features, stored as presummed textures.
        
    Returns:
        Interpolated feature vector.
    """
    # Determine the two resolution levels to interpolate between based on pixel_area_p
    # This logic is specific to the implementation but generally involves mapping p(x) to levels
    level_low, level_high = determine_levels_from_pixel_area(pixel_area_p)
    
    # Fetch presummed features from the lower resolution level
    # For 3D grids, use trilinear interpolation; for triplanes, use bilinear
    features_low = interpolate_texture(feature_grids, feature_triplanes, x, level_low)
    
    # Fetch presummed features from the higher resolution level
    features_high = interpolate_texture(feature_grids, feature_triplanes, x, level_high)
    
    # Interpolate between the two levels to get the final anti-aliased feature
    # The interpolation weight is determined by how close p(x) is to the boundaries of the levels
    weight = calculate_interpolation_weight(pixel_area_p, level_low, level_high)
    final_features = lerp(features_low, features_high, weight)
    
    return final_features

def interpolate_texture(grids, triplanes, x, level):
    """
    Interpolate features from the specified resolution level.
    """
    # Implementation depends on whether x falls within a 3D grid or a triplane
    # For 3D grids:
    if in_3d_grid(x, level):
        return trilinear_interpolate(grids[level], x)
    # For triplanes:
    else:
        return bilinear_interpolate(triplanes[level], x)
```
[G] The `determine_levels_from_pixel_area` and `calculate_interpolation_weight` functions are implementation-specific details not provided in the source. The core idea is interpolating between two presummed levels based on the sample's footprint [S1].

## Subtopics

### [CAwa-NeRF](11288-cawa-nerf.md)
A method for instant learning of compression-aware NeRF features that applies an entropy-aware training scheme with a rate-distortion trade-off to learn low entropy features, allowing export of zip-compressed feature grids with negligible overhead without changing the storage architecture or parameters of the base INGP method.

## Operating characteristics
- **Time Complexity:** The number of texture fetches per MLP evaluation is reduced to 8, compared to 24 (3+3×7) in the original hash-table-based InstantNGP approach, representing a 3× reduction [S1].
- **Memory Complexity:** Uses explicit 3D grids and triplanes, which have a fixed memory footprint based on resolution, unlike hash tables which can have variable overhead [S1].
- **Performance:** Benefits from hardware-accelerated texture interpolation and improved memory locality [S1].
- **Strengths:** Reduced fetch overhead, better memory locality, suitability for real-time rendering [S1].
- **Limitations:** Fixed memory usage, potential for interpolation artifacts if not carefully tuned [S1].
## Background
This method is part of the HybridNeRF architecture, which combines neural rendering with adaptive volumetric surfaces [S1]. It was introduced to optimize real-time rendering performance by leveraging explicit grid structures instead of hash tables [S1].
## Sources
[S1] HybridNeRF: Efficient Neural Rendering via Adaptive Volumetric Surfaces, https://doi.org/10.1109/cvpr52733.2024.01858, fetched 2026-10-26
