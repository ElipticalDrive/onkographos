# Point Feature Generation


## Overview
Point Feature Generation constructs per-point feature vectors for ray sampling locations by indexing pre-computed volume features and integrating multi-view image data [S1, unofficial]. It solves the problem of enriching sparse ray samples with high-dimensional appearance and geometric context, serving as the bridge between raw image inputs and volumetric rendering pipelines [S1, unofficial]. It differs from sibling methods by focusing on dynamic feature aggregation for arbitrary ray points rather than static voxelization or out-of-core storage [S1, unofficial][S2].

## How it works
**Inputs:**
*   `ray_points`: Set of 3D points sampled along rays.
*   `img_feat`: Optional multi-view image feature maps.
*   `input_feat`: Pre-computed volume features (e.g., from a neural field or grid).
*   `intrinsics`, `w2cs`, `c2ws`: Camera parameters for all input views.
*   `proj_mats`: Projection matrices for remapping features.

**Outputs:**
*   `ray_feats`: Final feature vectors for each ray point.

**Data Structures:**
*   `feat_dim`: Integer, total dimensionality of the output feature vector.
*   `color_volume`: 3D grid or hash table storing color features indexed by 3D coordinates.
*   `index_volume`: 3D grid or hash table storing geometric/structural features.

**Steps:**
1.  **Dimension Calculation:** If `img_feat` is provided, calculate `feat_dim` as the sum of the spatial dimensions of `img_feat` and any base feature dimensions [S1, unofficial].
2.  **Initialization:**
    *   If not using a color volume, initialize `input_feat` and `index_volume` [S1, unofficial].
    *   If using a color volume, prepare to directly index volume features [S1, unofficial].
3.  **Feature Assignment:**
    *   Assign the first 8 channels of `input_feat` to `ray_feats` [S1, unofficial].
    *   Assign the remaining channels to the `color_volume` built from images [S1, unofficial].
4.  **Indexing:**
    *   If using a color volume, directly index the volume features for the query points [S1, unofficial].
    *   Otherwise, aggregate features from the `index_volume` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_point_features(ray_points, img_feat, input_feat, intrinsics, w2cs, c2ws, proj_mats):
    """
    Generates features for ray points by indexing volume features and building color volumes.
    """
    # Step 1: Determine feature dimension
    feat_dim = 0
    if img_feat is not None:
        # Add spatial dimensions of img_feat to feat_dim
        # Assuming img_feat is a list of tensors or a 4D tensor [num_views, C, H, W]
        feat_dim += img_feat.shape[1]  # Channel dimension
    
    # Step 2: Initialize structures
    # If not using color volume, initialize input_feat and index volume features
    # [G] Assuming input_feat is already populated or passed in
    index_volume = None
    color_volume = None
    
    if img_feat is not None:
        # Build color volume from images
        # [G] This involves projecting img_feat to 3D space using camera parameters
        color_volume = build_color_volume(img_feat, intrinsics, w2cs, c2ws, proj_mats)
    
    # Step 3: Assign channels
    # Assign the first 8 channels of input_feat to ray_feats
    ray_feats = input_feat[:, :8]  # Assuming input_feat is [N, D]
    
    # Assign the remaining channels to the color volume built from images
    remaining_channels = input_feat[:, 8:]
    
    # Step 4: Indexing
    if color_volume is not None:
        # If using color volume, directly index volume features
        # [G] Sample color_volume at ray_points coordinates
        sampled_color = sample_volume(color_volume, ray_points)
        ray_feats = concatenate(ray_feats, sampled_color)
    else:
        # If not using color volume, index index_volume features
        # [G] Sample index_volume at ray_points coordinates
        sampled_index = sample_volume(index_volume, ray_points)
        ray_feats = concatenate(ray_feats, sampled_index)
        
    return ray_feats
```

**Parameters:**
*   `feat_dim`: Not given in the sources.
*   `num_channels`: Typically 8 for the initial assignment [S1, unofficial].

**Edge Cases:**
*   **Missing Depth:** If depth maps are unavailable, feature projection may fail [S1, unofficial].
*   **Occlusion:** Features from occluded views may be incorrect; attention mechanisms can help mitigate this [S2].

## Subtopics

### [Voxel Feature Composition](7021-voxel-feature-composition.md)
Voxel Feature Composition generates per-voxel feature vectors containing coordinates, RGB color, and opacity by sampling points within each Gaussian's volume and weighting them by Gaussian attenuation. It aggregates voxel-level geometric and appearance data into a single feature vector per voxel for storage or further processing. This method serves as the bridge between continuous 3D Gaussian Splatting representations and discrete voxel grids, enabling efficient embedding of high-dimensional features for quantile rendering.

### Attention Aggregation for 3D Points
Attention Aggregation for 3D Points aggregates unordered, variable-sized sets of retrieved pixel features from multiple input views into a single unique feature vector for a query 3D point, handling visual occlusions and arbitrary input counts [S2]. The attention mechanism must be permutation invariant with regard to the input set of feature vectors [S2]. For a query point $p$ and retrieved features $\{F^m_p\}$, it first generates position-aware features $\hat{F}^m_p = MLPs(F^m_p \oplus [x_p, y_p, z_p])$, then aggregates them using an attention function $A$: $\bar{F}_p = A(\{\hat{F}^m_p\})$ [S2].

## Operating characteristics
*   **Time Complexity:** Depends on the number of views and points. Indexing is $O(N)$ for $N$ points if using hash grids [S1, unofficial]. Attention aggregation is $O(M^2)$ or $O(M \log M)$ for $M$ views per point [S2].
*   **Memory Complexity:** Proportional to the size of the volume features and color volumes [S1, unofficial].
*   **Strengths:** Handles multi-view consistency via attention [S2]; flexible feature dimensionality [S1, unofficial].
*   **Limitations:** Requires pre-computed features or images [S1, unofficial]; sensitive to occlusions without attention [S2].
*   **When to use:** When ray points need rich, multi-view context [S1, unofficial][S2].
*   **When not to use:** When memory is extremely constrained or views are sparse [S2].

## Background
Point Feature Generation originated in the context of neural radiance fields and point-based rendering, specifically in methods like PointNeRF [S1, unofficial] and GRF [S2]. It addresses the need for efficient feature representation for 3D points sampled during volume rendering.

## Sources
[S1] Xharlie/pointnerf, https://github.com/Xharlie/pointnerf, fetched 2026-10-26 (unofficial)
[S2] GRF: Learning a General Radiance Field for 3D Representation and Rendering, https://doi.org/10.1109/iccv48922.2021.01490, fetched 2026-10-26
