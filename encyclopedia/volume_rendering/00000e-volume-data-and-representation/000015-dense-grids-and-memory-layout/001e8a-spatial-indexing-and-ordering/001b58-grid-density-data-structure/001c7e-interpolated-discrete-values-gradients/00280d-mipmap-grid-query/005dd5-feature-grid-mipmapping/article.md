# Feature Grid Mipmapping

Also called: Level-of-detail feature weighting, Level-of-detail for volumetric objects, Mip-mapped features

## Overview
Feature Grid Mipmapping is a level-of-detail (LOD) technique for neural radiance fields that improves rendering quality by eliminating aliasing artifacts in distant areas and revealing finer details in high-resolution regions [S1]. It addresses the performance and visual degradation issues inherent in querying high-resolution feature grids for objects at varying distances, particularly in virtual reality (VR) where dynamic camera movement exacerbates aliasing [S2, unofficial]. Unlike non-mip-mapped features that may bake in dynamic shadows or fail to model color correctly, this method uses implicit lowpass filtering to ensure cleaner geometry and appearance [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A set of hierarchical feature grids (levels $0$ to $L-1$), where level $0$ is the full-resolution grid and higher levels are progressively downsampled. The current viewpoint position $\mathbf{v}$.
*   **Output:** Interpolated feature values $\mathbf{f}$ at a query point $\mathbf{p}$.

**Data Structures**
The method relies on a hierarchy of 3D grids. Level $l$ has resolution $R_l \times R_l \times R_l$. The source notes that the parent method [Mipmap Grid Query](4154-mipmap-grid-query.md) assumes AABBs of multiple grids are 2x scaled, implying a geometric progression in spatial extent or resolution.

**Steps**
1.  **Grid Construction:** Create a hierarchy of feature grids by trilinearly downsampling the full-resolution (level 0) 3D feature grid to multiple lower-resolution levels [S2, unofficial].
2.  **Level Selection:** Determine the appropriate grid levels based on the distance $d$ from the viewpoint $\mathbf{v}$ to the query point $\mathbf{p}$ (or the object's center). Adaptively pick the two grid levels, $l_1$ and $l_2$, that are appropriate for this distance [S2, unofficial].
3.  **Querying:** Query the feature values $\mathbf{f}_{l_1}$ and $\mathbf{f}_{l_2}$ at point $\mathbf{p}$ from the selected grids.
4.  **Interpolation:** Linearly interpolate the feature values from the two selected grid levels to produce the final feature vector $\mathbf{f}$ [S2, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_feature_mipmap(grids, viewpoint, point):
    """
    grids: List of 3D feature grids, where grids[0] is full resolution.
    viewpoint: 3D vector, camera position.
    point: 3D vector, query position in world space.
    """
    # Calculate distance from viewpoint to the point
    # Note: The specific formula for distance-based level selection is not given in sources.
    # We assume a function that maps distance to a float level index.
    distance = vector_length(point - viewpoint)
    float_level = distance_to_level_index(distance) # [G] Placeholder for specific LOD logic
    
    # Pick the two adjacent integer levels
    level_low = floor(float_level)
    level_high = ceil(float_level)
    
    # Clamp levels to valid range [0, num_levels - 1]
    level_low = max(0, min(level_low, len(grids) - 1))
    level_high = max(0, min(level_high, len(grids) - 1))
    
    # Query features from both levels
    # Trilinear interpolation is assumed for each grid query
    features_low = trilinear_interpolate(grids[level_low], point)
    features_high = trilinear_interpolate(grids[level_high], point)
    
    # Linearly interpolate between the two levels
    # The blend factor depends on the fractional part of the float_level
    # [G] Standard mipmap blending logic
    t = float_level - floor(float_level)
    features = lerp(features_low, features_high, t)
    
    return features
```

**Parameters**
*   **Number of Levels:** Not given in the sources.
*   **Downsampling Factor:** Trilinear downsampling is used [S2, unofficial]. The parent method [Mipmap Grid Query](4154-mipmap-grid-query.md) implies 2x scaling relationships.

**Edge Cases and Pitfalls**
*   **Aliasing:** Severe aliasing artifacts can occur in VR due to varying distances. This method eliminates them via implicit lowpass filtering [S2, unofficial].
*   **Dynamic Shadows:** Non-mip-mapped features may bake in dynamic shadows from the rig in training data, which are modeled by high-frequency levels and addressed by this distance-aware approach [S1].
*   **Capacity Allocation:** High-resolution hash grids are specifically allocated to areas rich in fine details in training views, allowing the model to automatically allocate more capacity for these parts [S1].

## Operating characteristics
*   **Performance:** Querying high-resolution feature grids can become expensive with many visible objects. This method makes feature querying for far-away objects faster by using lower-resolution grids [S2, unofficial].
*   **Visual Quality:** Eliminates aliasing for distant areas and reveals more detail compared to non-mip-mapped features [S1]. Provides cleaner geometry and appearance in ambiguous spaces [S1].
*   **VR Suitability:** Critical for high-fidelity rendering results that contribute to a pleasant VR experience, achieving significantly higher resolution, frame rate, and visual fidelity required for comfortable VR viewing [S1].

## Background
This method was introduced in the context of VR-NeRF and Neural Assets for high-fidelity virtualized walkable spaces and interactive environments [S1][S2, unofficial]. It addresses the challenges of rendering neural radiance fields in VR, where comfortable viewing requires high frame rates and visual fidelity [S1]. The technique is part of a pipeline that includes a multi-camera rig for capturing HDR photos and a perceptual color space for HDR model optimization [S1].

## Sources
[S1] VR-NeRF: High-Fidelity Virtualized Walkable Spaces, https://doi.org/10.1145/3610548.3618139, fetched 2026-10-28
[S2] Neural Assets: Volumetric Object Capture and Rendering for Interactive Environments, https://core.ac.uk/download/552466290.pdf, fetched 2026-10-30 (unofficial)
