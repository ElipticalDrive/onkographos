# Cross-View Feature Matching and Cost Volume Construction

Also called: Cost Volume Construction, Multi-View Feature Matching, Multi-View Feature Matching with Cost Volume, Plane-Sweep Cost Volume

## Overview
This method constructs per-view cost volumes from multiple 2D image features to enable robust depth estimation and 3D feature lifting in multi-view reconstruction [S1, S2]. It differs from sibling methods like [Attention-based Multi-View Density Prediction](10674-attention-based-multi-view-density-prediction.md) by explicitly using a plane-sweep strategy to aggregate geometric consistency across views, rather than relying solely on attention mechanisms for density prediction [S1, S2]. It builds on [Encoder and Feature Fusion](7178-encoder-and-feature-fusion.md) by taking refined cross-view features as input [S1, S2].

## How it works
**Inputs:**
- $N$ input RGB images.
- Camera intrinsics and extrinsics for each view.

**Outputs:**
- Per-view cost volumes $\{C_i\}_{i=1}^N$, where $C_i \in \mathbb{R}^{\frac{H}{s} \times \frac{W}{s} \times D}$ [S1, S2].
- Refined cross-view features $\{F_i\}_{i=1}^N$, where $F_i \in \mathbb{R}^{\frac{H}{s} \times \frac{W}{s} \times C}$ [S1, S2].
- Predicted depth maps $\{D_i\}_{i=1}^N$, where $D_i \in \mathbb{R}^{H \times W}$ [S1, S2].

**Data Structures:**
- Feature maps: 4D tensors stored in memory as contiguous arrays [S1, S2].
- Cost volume: 3D tensor where the third dimension represents depth [S1, S2].
- Depth map: 2D tensor of float values [S1, S2].

**Steps:**

1. **Feature Extraction:**
   Apply a weight-sharing ResNet backbone to each of the $N$ input RGB images to obtain $s\times$ downsampled feature maps [S1, S2].

2. **Cross-View Refinement:**
   Refine the extracted features using cross-view attention that exchanges information with the two nearest neighboring views using local window attention [S1, S2].

3. **Plane-Sweep Cost Volume Construction:**
   For each reference view $i$:
   a. Sample $D$ candidate depths $\{d_m\}_{m=1}^D$ [S1, S2].
   b. For each depth $d_m$ and each of the two selected neighboring views $j$:
      i. Warp the feature map $F_j$ to the reference view $i$'s coordinate system at hypothesized depth $d_m$ using the camera projection matrix [S1, S2].
      ii. Compute pairwise feature similarity between the warped features and the reference feature $F_i$ using the dot-product operation [S1, S2].
   c. Combine the correlation values for the two selected views with averaging [S1].
   d. Stack all correlations along the depth axis to form the cost volume $C_i \in \mathbb{R}^{\frac{H}{s} \times \frac{W}{s} \times D}$ [S1, S2].

4. **Depth Regression:**
   Concatenate the monocular feature $F_{\text{mono},i}$ and the cost volume $C_i$ in the channel dimension [S1]. Pass the concatenated tensor through a 2D U-Net to regress depth [S1, S2]. Normalize the $D$ dimension with the softmax operation and perform a weighted average of all depth candidates using the softmax probabilities to obtain the final depth output [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import numpy as np

def apply_resnet(image):
    """Standard weight-sharing ResNet backbone. Returns downsampled feature map."""
    # Returns tensor of shape (H/s, W/s, C)
    return np.zeros((image.shape[0] // 4, image.shape[1] // 4, 256))

def get_nearest_neighbors(ref_idx, cameras, k=2):
    """Finds k nearest neighboring camera centers."""
    ref_center = cameras[ref_idx].center
    distances = [np.linalg.norm(cam.center - ref_center) for cam in cameras]
    distances[ref_idx] = np.inf
    sorted_indices = np.argsort(distances)
    return sorted_indices[:k]

def local_window_attention(ref_feat, neighbor_feats, window_size):
    """Local window attention mechanism."""
    combined = np.concatenate([ref_feat] + neighbor_feats, axis=-1)
    return combined[..., :ref_feat.shape[-1]]

def warp_feature(neighbor_feat, neighbor_cam, ref_cam, depth_hypothesis):
    """Warp feature from neighbor view to reference view at given depth."""
    # Uses camera intrinsics and extrinsics to project neighbor pixels to reference view
    return np.zeros((neighbor_feat.shape[0], neighbor_feat.shape[1], neighbor_feat.shape[2]))

def dot_product_similarity(ref_feat, warped_feat):
    """Compute pairwise feature similarity using dot-product."""
    ref_norm = ref_feat / (np.linalg.norm(ref_feat, axis=-1, keepdims=True) + 1e-8)
    warp_norm = warped_feat / (np.linalg.norm(warped_feat, axis=-1, keepdims=True) + 1e-8)
    return np.sum(ref_norm * warp_norm, axis=-1)

def aggregate_similarities(similarities_list):
    """Combine correlation values for selected views with averaging."""
    return np.mean(similarities_list, axis=0)

def depth_regression_module(ref_feat, cost_vol):
    """Fuse monocular features and cost volume, regress depth via U-Net."""
    fused = np.concatenate([ref_feat, cost_vol], axis=-1)
    depth_logits = np.zeros((fused.shape[0], fused.shape[1], cost_vol.shape[-1]))
    probs = np.exp(depth_logits) / np.sum(np.exp(depth_logits), axis=-1, keepdims=True)
    depth_candidates = np.linspace(0.1, 10.0, cost_vol.shape[-1])
    depth_map = np.sum(probs * depth_candidates, axis=-1)
    return depth_map

def cross_view_feature_matching_and_cost_volume_construction(images, cameras, s, C, D):
    N = len(images)
    features = [apply_resnet(img) for img in images]
    refined_features = []
    for i in range(N):
        neighbors = get_nearest_neighbors(i, cameras, k=2)
        neighbor_feats = [features[n] for n in neighbors]
        refined = local_window_attention(features[i], neighbor_feats, window_size=7)
        refined_features.append(refined)

    cost_volumes = []
    for i in range(N):
        ref_feat = refined_features[i]
        neighbors = get_nearest_neighbors(i, cameras, k=2)
        depths = np.linspace(0.1, 10.0, D)
        cost_vol = np.zeros((ref_feat.shape[0], ref_feat.shape[1], D))

        for m, d in enumerate(depths):
            similarities = []
            for j in neighbors:
                warped = warp_feature(refined_features[j], cameras[j], cameras[i], d)
                sim = dot_product_similarity(ref_feat, warped)
                similarities.append(sim)
            cost_vol[:, :, m] = aggregate_similarities(similarities)
        cost_volumes.append(cost_vol)

    depth_maps = []
    for i in range(N):
        depth_map = depth_regression_module(refined_features[i], cost_volumes[i])
        depth_maps.append(depth_map)

    return cost_volumes, refined_features, depth_maps
```

**Parameters:**
- $s$: Feature downsampling factor. $s=4$ for 256×256 resolution (RealEstate10K); $s=8$ for higher resolution datasets (TartanAir, ScanNet, KITTI, DL3DV) [S1].
- $C$: Feature dimension. Not given in the sources.
- $D$: Number of candidate depths. Not given in the sources.
- Window size for local attention. Not given in the sources.

**Edge Cases and Pitfalls:**
- Precision in warping features depends on accurate camera parameters [G].
- Memory usage grows linearly with $N$, $D$, and image resolution [G].
- Boundary handling for sequence ends or sparse views is implementation-dependent [G].

## Subtopics

### Variance-based Feature Aggregation
Projects each vertex $v$ of a bounding box to 2D feature maps $F_i$ using camera parameters to obtain features $F_i(v)$ by interpolation [S3]. Constructs a cost volume $B$ by computing the variance of the $N$ projected features for each vertex: $B(v) = \operatorname{Var}(\{F_i(v)\}_{i=0}^{N-1})$ [S3].

### Hierarchical Matching
Performs coarse depth prediction at a lower resolution (e.g., 1/8 or 1/4 of original image) [S1]. Upsamples the coarse depth prediction by 2× and performs correspondence search on feature maps with 2× higher resolution, restricted to neighbors of the upsampled coarse depth [S1]. Constructs a smaller cost volume for the fine stage because the search range is limited by the coarse prediction [S1]. Uses the same feature fusion and depth regression procedure as the coarse stage to get higher-resolution depth predictions, which are subsequently upsampled to full resolution using a DPT head [S1]. Uses 2-scale features, e.g., 1/8 and 1/4, or 1/4 and 1/2 resolutions [S1].

### Feature Fusion and Depth Regression
Concatenates the monocular feature $F_{\text{mono},i}$ and the cost volume $C_i$ in the channel dimension [S1]. Passes the concatenated tensor through a 2D U-Net to regress depth [S1]. Normalizes the $D$ dimension (depth candidates) with the softmax operation and performs a weighted average of all depth candidates using the softmax probabilities to obtain the final depth output [S1]. The U-Net output tensor has shape $(H/s, W/s, D)$ [S1].

### Adaptive Depth Hypothesis Sampling and Cost Volume Construction
Warps a pixel $p$ in the reference view to the $i$-th source view at depth plane hypothesis $d$ as $p_{i0} = K_i [(R_i (d K^{-1} p) + t_i)]$, where $K_i$ and $K$ are intrinsic matrices, and $R_i$ and $t_i$ are rotation and translation between views [S4]. Consolidates multiple feature volumes $\{V_i\}_{i=1}^N$ into a 3D cost volume $V$ using variance-based aggregation [S4]. Regularizes the cost volume into a depth probability volume using a 3D U-Net to determine probability $P_j(p)$ on depth plane $D_j(p)$ for pixel $p$ [S4]. Calculates the estimated depth value $\hat{D}(p) = \sum_{j=1}^J P_j(p) D_j(p)$ [S4].

## Operating characteristics
- **Time Complexity:** Proportional to $N \times D \times (H/s) \times (W/s)$ for warping and similarity computation [G].
- **Memory Complexity:** Proportional to $N \times (H/s) \times (W/s) \times D$ for storing cost volumes [G].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales efficiently to a larger number of input views by selecting only the top-2 nearest neighbors per reference view [S1].
- **Strengths:** Achieves a good speed-accuracy trade-off and produces robust, multi-view consistent depth estimates [S1, S2].
- **Limitations:** Computational cost of plane-sweep and warping increases with $D$ and resolution [G].
- **When to use:** Multi-view reconstruction requiring explicit geometric consistency and robust depth estimation [S1, S2].
- **When not to use:** Real-time applications with strict latency constraints if $D$ is large or resolution is high [G].

## Background
Originated in multi-view stereo (MVS) literature and adapted for feed-forward 3D Gaussian Splatting architectures [S1, S2]. The method is designed to decouple 3D representation from the 2D pixel grid [S2]. It uses plane-sweep strategy [S1, S2] and dot-product matching [S1, S2] as cited in the sources.

## Sources
[S1] DepthSplat: Connecting Gaussian Splatting and Depth, https://doi.org/10.1109/cvpr52734.2025.01534, fetched 2026-10-26
[S2] VolSplat: Rethinking Feed-Forward 3D Gaussian Splatting with Voxel-Aligned Prediction, https://doi.org/10.1007/978-3-032-37232-1_1, fetched 2026-10-26
[S3] SparseNeuS: Fast Generalizable Neural Surface Reconstruction from Sparse Views, https://doi.org/10.1007/978-3-031-19824-3_13, fetched 2026-10-26
[S4] Multi-View Stereo Network Based on Attention Mechanism and Neural Volume Rendering, https://doi.org/10.3390/electronics12224603, fetched 2026-10-26
