# Cross-View Feature Matching and Cost Volume Construction

Also called: Cost Volume Construction, Plane-Sweep Cost Volume

## Overview
This method constructs a 3D cost volume from multiple 2D image features to enable robust depth estimation and 3D feature lifting in multi-view reconstruction. It differs from sibling methods like [Attention-based Multi-View Density Prediction](10674-attention-based-multi-view-density-prediction.md) by explicitly using a plane-sweep strategy to aggregate geometric consistency across views, rather than relying solely on attention mechanisms for density prediction. It builds on [Encoder and Feature Fusion](7178-encoder-and-feature-fusion.md) by taking refined cross-view features as input.

## How it works
**Inputs:**
- $N$ input RGB images.
- Camera intrinsics and extrinsics for each view.
- Hyperparameters: downsampling factor $p$, feature dimension $C$, number of candidate depths $D$, window size for local attention.

**Outputs:**
- Per-view cost volumes $\{C_i\}_{i=1}^N$, where $C_i \in \mathbb{R}^{\frac{H}{p} \times \frac{W}{p} \times D}$.
- Refined cross-view features $\{F_i\}_{i=1}^N$, where $F_i \in \mathbb{R}^{\frac{H}{p} \times \frac{W}{p} \times C}$.
- Predicted depth maps $\{D_i\}_{i=1}^N$, where $D_i \in \mathbb{R}^{H \times W}$.

**Data Structures:**
- Feature maps: 4D tensors stored in memory as contiguous arrays.
- Cost volume: 3D tensor where the third dimension represents depth.
- Depth map: 2D tensor of float values.

**Steps:**

1. **Feature Extraction:**
   Apply a weight-sharing ResNet backbone [S1] to each of the $N$ input RGB images. This produces downsampled feature maps $F_{raw} \in \mathbb{R}^{\frac{H}{p} \times \frac{W}{p} \times C}$ for each view [S1].

2. **Cross-View Refinement:**
   Refine the extracted features using cross-view attention. For each view $i$, exchange information with its two nearest neighboring views using local window attention [S1]. This produces cross-view-aware features $F_i \in \mathbb{R}^{\frac{H}{p} \times \frac{W}{p} \times C}$ [S1].

3. **Plane-Sweep Cost Volume Construction:**
   For each reference view $i$:
   a. Sample $D$ candidate depths $\{d_m\}_{m=1}^D$ [S1].
   b. For each depth $d_m$ and each neighboring view $j$:
      i. Warp the feature map $F_j$ to the reference view $i$'s coordinate system at hypothesized depth $d_m$ using the known camera parameters [S1].
      ii. Compute pairwise feature similarity between the warped features and the reference feature $F_i$ [S1].
   c. Aggregate similarities using dot-product matching [S1].
   d. Stack the aggregated similarities along the depth axis to form the cost volume $C_i \in \mathbb{R}^{\frac{H}{p} \times \frac{W}{p} \times D}$ [S1].

4. **Depth Regression:**
   Fuse the monocular features $F_{mono}$ (derived from $F_i$) with the cost volume $C_i$ to regress a dense per-pixel depth map $D_i \in \mathbb{R}^{H \times W}$ [S1]. This depth map serves as a geometric prior for subsequent 3D lifting [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cross_view_feature_matching_and_cost_volume_construction(images, cameras, p, C, D, window_size):
    """
    images: List of N RGB images, shape (H, W, 3)
    cameras: List of N camera objects with intrinsics and extrinsics
    p: Downsampling factor
    C: Feature dimension
    D: Number of candidate depths
    window_size: Size for local window attention
    """
    N = len(images)
    features = []
    
    # Step 1: Feature Extraction
    for i in range(N):
        # Apply weight-sharing ResNet backbone
        # Output shape: (H/p, W/p, C)
        feat = resnet_backbone(images[i])
        features.append(feat)
        
    # Step 2: Cross-View Refinement
    refined_features = []
    for i in range(N):
        # Identify two nearest neighboring views
        neighbors = get_nearest_neighbors(i, cameras)
        
        # Apply local window attention with neighbors
        # Input: features[i] and features[neighbors]
        # Output: refined feature for view i
        ref_feat = local_window_attention(features[i], [features[n] for n in neighbors], window_size)
        refined_features.append(ref_feat)
        
    # Step 3: Plane-Sweep Cost Volume Construction
    cost_volumes = []
    for i in range(N):
        ref_feat = refined_features[i]
        ref_cam = cameras[i]
        
        # Sample D candidate depths
        depths = sample_candidate_depths(D)
        
        # Initialize cost volume tensor: (H/p, W/p, D)
        cost_vol = zeros((ref_feat.shape[0], ref_feat.shape[1], D))
        
        for m, d in enumerate(depths):
            similarities = []
            # Get neighboring views for warping
            neighbors = get_nearest_neighbors(i, cameras)
            
            for j in neighbors:
                neighbor_feat = refined_features[j]
                neighbor_cam = cameras[j]
                
                # Warp neighbor feature to reference view at depth d
                # Uses camera intrinsics and extrinsics
                warped_feat = warp_feature(neighbor_feat, neighbor_cam, ref_cam, d)
                
                # Compute pairwise feature similarity (dot-product)
                # Between ref_feat and warped_feat
                sim = dot_product_similarity(ref_feat, warped_feat)
                similarities.append(sim)
                
            # Aggregate similarities (e.g., mean or sum)
            aggregated_sim = aggregate_similarities(similarities)
            
            # Stack along depth axis
            cost_vol[:, :, m] = aggregated_sim
            
        cost_volumes.append(cost_vol)
        
    # Step 4: Depth Regression
    depth_maps = []
    for i in range(N):
        ref_feat = refined_features[i]
        cost_vol = cost_volumes[i]
        
        # Fuse monocular features with cost volume
        # Regress dense per-pixel depth map
        # Output shape: (H, W)
        depth_map = depth_regression_module(ref_feat, cost_vol)
        depth_maps.append(depth_map)
        
    return cost_volumes, refined_features, depth_maps
```

**Parameters:**
- $p$: Downsampling factor. Typical value not specified in sources.
- $C$: Feature dimension. Not specified in sources.
- $D$: Number of candidate depths. Not specified in sources.
- Window size: For local window attention. Not specified in sources.

**Edge Cases and Pitfalls:**
- Precision in warping features depends on accurate camera parameters.
- Degenerate input: If neighboring views are not available (e.g., at sequence ends), handle gracefully [S1].
- Memory usage grows with $D$ and $N$.

## Subtopics

### [Low-cost Cross-View Aggregation](15153-low-cost-cross-view-aggregation.md)
Low-cost Cross-View Aggregation is a module for efficient feature extraction and matching among nearby views to predict initial Gaussian triplets, enabling long sequence reconstruction without computationally intractable transformers. It sets itself apart from sibling methods by using adaptive cost volumes and multi-scale aggregation to broaden the receptive field, allowing for the processing of more nearby views at low computational cost. See [Low-cost Cross-View Aggregation](15153-low-cost-cross-view-aggregation.md).

### Depth Module for Geometric Prior
The depth map $D_i$ serves as a geometric prior for lifting image features into 3D space [S1]. This module fuses monocular features with the cost volume to regress dense per-pixel depth maps, which are then used to construct 3D point clouds and voxel-based features for volumetric reasoning [S1].

## Operating characteristics
- **Time Complexity:** Depends on $N$, $D$, and the cost of warping and similarity computation. Not explicitly given.
- **Memory Complexity:** Proportional to $N \times \frac{H}{p} \times \frac{W}{p} \times D$ for cost volumes.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with number of views and candidate depths.
- **Strengths:** Robust multi-view consistent depth estimates [S1].
- **Limitations:** Computational cost of plane-sweep and warping.
- **When to use:** Multi-view reconstruction requiring explicit geometric consistency.
- **When not to use:** Real-time applications with strict latency constraints if $D$ is large.

## Background
Originated in VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction [S1]. The method is designed to decouple 3D representation from the 2D pixel grid [S1]. It uses plane-sweep strategy [57] and dot-product matching [9] as cited in [S1].

## Sources
[S1] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
