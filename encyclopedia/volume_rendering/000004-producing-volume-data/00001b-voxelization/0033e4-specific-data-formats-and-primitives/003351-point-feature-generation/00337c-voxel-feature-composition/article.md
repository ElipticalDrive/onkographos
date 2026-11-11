# Voxel Feature Composition

Also called: Gaussian Voxel Feature Generation, compose-voxel-features

## Overview
Voxel Feature Composition generates per-voxel feature vectors by sampling points within the volume of 3D Gaussians and weighting them by Gaussian attenuation [S1, unofficial]. It aggregates voxel-level geometric and appearance data into a single feature vector per voxel for storage or further processing [S1, unofficial]. Unlike its sibling [Mean Feature Aggregation by Voxel](11242-mean-feature-aggregation-by-voxel.md), which relies on direct spatial averaging of raw LiDAR points, this method computes features by sampling Gaussian volumes and weighting by attenuation [S1, unofficial].

## How it works
**Inputs:**
- `gaussians`: A list of Gaussian definitions. Each Gaussian dictionary contains:
  - `mean`: 3D vector $\mu$ (Gaussian mean).
  - `inverse_3d_covariance_matrix`: $3 \times 3$ matrix $\Sigma^{-1}$ (inverse covariance).
  - `opacity`: Scalar $\alpha$ (Gaussian opacity factor).
  - `rgb`: 3D vector (color).
- `num_voxels_per_g`: List or tensor of integers, specifying the number of voxels associated with each Gaussian.
- `grid_size`: Float, the size of each voxel cell.

**Outputs:**
- `voxel_xyzs`: List of 3D coordinate tensors for all generated voxels.
- `voxel_features`: List of feature tensors. Each tensor has shape $[N, 7]$, where $N$ is the number of voxels for that Gaussian. The 7 dimensions are: 3 for position ($xyz$), 3 for color ($rgb$), and 1 for opacity.

**Data Structures:**
- The feature vector is a concatenation along dimension 1: `[voxel_xyz, color_rgb, voxel_opacity]`.

**Steps:**
1. Initialize empty lists `voxel_xyzs` and `voxel_features`.
2. Iterate over each Gaussian $g$ in `gaussians` and its corresponding voxel count $N$ from `num_voxels_per_g`.
3. Extract parameters from $g$:
   - $\mu = g[\text{"mean"}]$
   - $\Sigma^{-1} = g[\text{"inverse_3d_covariance_matrix"}]$
   - $\alpha = g[\text{"opacity"}]$
   - $c = g[\text{"rgb"}]$
4. For each voxel index $i$ from $0$ to $N-1$:
   a. Compute the 3D coordinate $v$ of the voxel using `get_voxel_location(i, g, grid_size)` [S1, unofficial].
   b. Compute the Mahalanobis distance:
      $$dist = (v - \mu)^T \Sigma^{-1} (v - \mu)$$
      where $v$ is the voxel coordinate, $\mu$ is the Gaussian mean, and $\Sigma^{-1}$ is the inverse covariance matrix [S1, unofficial].
   c. Compute the voxel opacity:
      $$opacity = \alpha \cdot \exp(-0.5 \cdot dist)$$
      where $\alpha$ is the Gaussian opacity factor and $dist$ is the Mahalanobis distance [S1, unofficial].
   d. Concatenate features:
      $$feature\_vec = [v, c, opacity]$$
   e. Append $v$ to `voxel_xyzs` and $feature\_vec$ to `voxel_features`.
5. Return `voxel_xyzs` and `voxel_features`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
from typing import List, Dict, Tuple

def get_voxel_location(voxel_idx: int, gaussian: Dict, grid_size: float) -> torch.Tensor:
    """
    [G] Placeholder for implementation detail not fully specified in source.
    Returns the 3D coordinate tensor for a specific voxel index within a Gaussian's bounding volume.
    """
    # Implementation depends on specific voxelization strategy (e.g., bounding box sampling).
    # The source assumes this function exists and returns a tensor of shape [3].
    raise NotImplementedError("Implementation of get_voxel_location is external to the core formula.")

def compose_voxel_features(
    gaussians: List[Dict], 
    num_voxels_per_g: List[int], 
    grid_size: float
) -> Tuple[List[torch.Tensor], List[torch.Tensor]]:
    """
    Generate per-voxel feature vectors for a set of 3D Gaussians.
    """
    voxel_xyzs = []
    voxel_features = []
    
    for gaussian, num_voxels in zip(gaussians, num_voxels_per_g):
        mu = gaussian["mean"]
        inv_cov = gaussian["inverse_3d_covariance_matrix"]
        opacity_factor = gaussian["opacity"]
        color_rgb = gaussian["rgb"]
        
        for voxel_idx in range(num_voxels):
            # 1. Get the 3D coordinate for the current voxel
            voxel_xyz = get_voxel_location(voxel_idx, gaussian, grid_size)
            voxel_xyzs.append(voxel_xyz)
            
            # 2. Compute the Mahalanobis distance
            # dist = (v - mu)^T @ inv_cov @ (v - mu)
            diff = voxel_xyz - mu
            dist = torch.dot(diff, torch.matmul(inv_cov, diff))
            
            # 3. Compute voxel opacity using a Gaussian attenuation factor
            voxel_opacity = opacity_factor * torch.exp(-0.5 * dist)
            
            # 4. Concatenate features: [xyz, rgb, opacity]
            # Ensure tensors are 1D for concatenation
            feature_vec = torch.cat([voxel_xyz, color_rgb, voxel_opacity.unsqueeze(0)], dim=0)
            voxel_features.append(feature_vec)
            
    return voxel_xyzs, voxel_features
```

**Parameters and Edge Cases:**
- `grid_size`: Determines the spatial resolution. Not given in the sources.
- `num_voxels_per_g`: Precomputed count. If zero, the Gaussian contributes no voxels.
- Precision: The Mahalanobis distance calculation involves matrix-vector multiplication. Ensure `inv_cov` is positive definite to avoid numerical instability in the exponential term [S1, unofficial].

## Subtopics

### [Mean Feature Aggregation by Voxel](11242-mean-feature-aggregation-by-voxel.md)
Computes mean features for each voxel by aggregating point features based on voxel indices. It serves as a geometric feature extraction step that converts sparse, noisy LiDAR point clouds into dense, structured neural volumetric features. Unlike Voxel Feature Composition, it relies on direct spatial averaging of raw LiDAR points followed by a 3D sparse convolutional network to encode global context.

## Operating characteristics
- **Time Complexity:** Linear with respect to the total number of voxels generated across all Gaussians. Each voxel requires a Mahalanobis distance calculation ($O(1)$ for fixed 3D dimensions) and concatenation.
- **Memory Complexity:** Proportional to the total number of voxels. Each voxel stores a 7-element feature vector.
- **Strengths:** Provides a structured, dense representation of Gaussian splats suitable for downstream neural networks (e.g., sparse convolutions) [S1, unofficial].
- **Limitations:** Requires precomputing the number of voxels per Gaussian and a helper function to map voxel indices to 3D coordinates, which is not detailed in the source [S1, unofficial].
- **When to use:** When preparing 3D Gaussian Splatting data for quantile rendering or semantic segmentation tasks that require voxelized inputs [S1, unofficial].

## Background
Voxel Feature Composition was introduced in the context of "Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting" [S1, unofficial]. It is used to bridge the gap between continuous 3D Gaussian representations and discrete voxel-based neural processing pipelines, enabling efficient embedding of high-dimensional features [S1, unofficial].

## Sources
[S1] Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2512.20927, fetched 2026-10-26 (unofficial)
