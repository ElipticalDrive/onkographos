# Multi-scale Feature Interpolation

Also called: HexPlaneField Interpolation

## Overview
Multi-scale feature interpolation combines feature representations from multiple resolution levels to reconstruct dense 3D features at query points [S1, unofficial]. It addresses single-resolution limitations by aggregating coarse structural information with fine-grained details [S1, unofficial]. Unlike standard interpolation methods that operate on a single fixed grid, this method explicitly manages a hierarchy of grid scales, concatenating or accumulating interpolated results across these levels [S1, unofficial].

## How it works
**Inputs:**
- `pts`: Query points tensor. Shape and dimensions are not given in the sources. [G] Standard practice assumes $(N, 3)$ for 3D coordinates.
- `ms_grids`: List of grid tensors, one per scale level. Layout details are not given in the sources. [G] Each grid typically holds spatial features at a specific resolution.
- `num_levels`: Integer specifying the number of scales in `ms_grids`. [G]
- `concat_features`: Boolean flag determining the fusion strategy. [S1, unofficial]

**Outputs:**
- `multi_scale_interp`: Interpolated feature tensor. Shape depends on `concat_features`. [G] If `True`, features are concatenated along the channel dimension. If `False`, they are summed.

**Data Structures:**
- The method operates on a list of grid tensors. The specific HexPlane decomposition into six 2D planes is not detailed in the provided source text. [S1, unofficial] The implementation relies on a per-scale interpolation step that maps query coordinates to grid indices and evaluates basis functions.

**Steps:**
1. Initialize an empty list `multi_scale_interp` to store results from each scale.
2. Iterate through each scale level $l$ from $0$ to `num_levels` - 1:
   a. Generate all combinations of coordinates of size `grid_dimensions` from the input point dimensions (`range(pts.shape[-1])`). [S1, unofficial]
   b. Initialize `interp_space` to $1.0$. [S1, unofficial]
   c. Compute the interpolated features for the query points `pts` using the grid $\mathbf{G}_l$. [G]
   d. If `concat_features` is `True`, append `interp_space` to `multi_scale_interp`. [S1, unofficial]
   e. If `concat_features` is `False`, add `interp_space` to the accumulated tensor. [S1, unofficial]
3. If `concat_features` is `True`, concatenate all tensors in `multi_scale_interp` along the last dimension (`dim=-1`). [S1, unofficial]
4. Return the final tensor.

**Formulas:**
- Accumulation (when `concat_features` is `False`): $I_{\text{acc}} = \sum_{l=0}^{L-1} I_l$
- Concatenation (when `concat_features` is `True`): $I_{\text{out}} = \text{concat}(I_0, I_1, \dots, I_{L-1}, \text{dim}=-1)$

**Parameters:**
- `concat_features`: Default `True`. Determines the fusion strategy. [S1, unofficial]

**Edge Cases and Pitfalls:**
- **Channel Mismatch:** If `concat_features` is `False`, the channel dimensions of all scales must be identical to allow addition. [G]
- **Grid Resolution:** Coarse grids may lack high-frequency detail, while fine grids may be noisy. [G]
- **Coordinate Normalization:** Query coordinates must be normalized to the grid's indexing space consistent with grid construction. [G]

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def interpolate_ms_features(pts, ms_grids, num_levels, concat_features=True):
    """
    Interpolates features from multiple grid scales.
    
    Args:
        pts: Tensor of query coordinates.
        ms_grids: List of grid tensors, one per scale level.
        num_levels: Integer, number of scales.
        concat_features: Boolean, whether to concatenate or sum features.
        
    Returns:
        Tensor of interpolated features.
    """
    multi_scale_interp = []
    accumulated_features = None
    
    for l in range(num_levels):
        grid = ms_grids[l]
        
        # Generate coordinate combinations from input point dimensions [S1, unofficial]
        # Initialize interpolation space [S1, unofficial]
        interp_space = 1.0
        
        # Compute interpolated features for query points using the grid [G]
        # The specific interpolation kernel is not detailed in the sources [G]
        # Standard implementation maps pts to grid indices and evaluates basis functions
        interp_space = interp_space  # Placeholder for actual kernel application [G]
        
        if concat_features:
            multi_scale_interp.append(interp_space)
        else:
            if accumulated_features is None:
                accumulated_features = interp_space
            else:
                accumulated_features = accumulated_features + interp_space
                
    if concat_features:
        # Concatenate along the feature dimension (last dim) [G]
        result = concat(multi_scale_interp, dim=-1)
    else:
        result = accumulated_features
        
    return result
```

## Subtopics

### 3D Gaussian Feature Interpolation
This variant interpolates features using pre-computed 3D Gaussian parameters rather than a dense grid. It computes Gaussian weights based on the distance between query points and Gaussian centers, weighted by the inverse covariance matrix, and performs a weighted sum of features. [S2, unofficial] The data structure includes feature tensors `feats` of shape $(B, C, K)$, offsets `offsets` of shape $(B, K, 3)$, determinants `det` of shape $(B, K)$, and inverse covariance matrices `inv` of shape $(B, K, 3, 3)$. [S2, unofficial] It is chosen when explicit geometric primitives are preferred over voxel-based representations. [G]

## Operating characteristics
- **Time Complexity:** Not given in the sources. [G] Standard multi-scale interpolation scales linearly with the number of query points $N$, scales $L$, and interpolation cost $K$.
- **Memory Complexity:** Not given in the sources. [G] Storage scales with the sum of grid dimensions and channel counts across all levels. Concatenation increases output memory by a factor of $L$ if `concat_features` is `True`. [G]
- **Performance:** Training time is reported as ~8 minutes for synthetic scenes (D-NeRF) and ~30 minutes for real dynamic scenes (HyperNeRF). [S1, unofficial]
- **Strengths:** Captures multi-resolution details; effective for dynamic scenes. [G]
- **Limitations:** Requires careful tuning of scale levels; concatenation increases feature dimensionality, potentially impacting downstream decoder complexity. [G]

## Background
Multi-scale feature interpolation using HexPlane grids was introduced in the context of 4D Gaussian Splatting for dynamic scene representation. The method was part of the 4DGaussians framework, accepted at CVPR 2024. [S1, unofficial]

## Sources
[S1] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
[S2] xmed-lab/DIF-Gaussian, https://github.com/xmed-lab/DIF-Gaussian, fetched 2026-10-27 (unofficial)
