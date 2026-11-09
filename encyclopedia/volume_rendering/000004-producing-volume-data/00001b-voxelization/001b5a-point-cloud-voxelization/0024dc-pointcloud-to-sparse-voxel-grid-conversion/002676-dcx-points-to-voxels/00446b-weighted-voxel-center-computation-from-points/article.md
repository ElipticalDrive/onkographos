# Weighted Voxel Center Computation from Points

Also called: Weighted Mean Voxel Center

## Overview
This method computes the spatial center of each voxel in a sparse voxel grid by calculating the weighted mean of 3D points falling within that voxel, using depth confidence as weights [S1, unofficial]. It serves as the final aggregation step in the DCX Points-to-Voxels pipeline, refining the raw point cloud into a structured sparse representation suitable for downstream rendering or reconstruction tasks [S1, unofficial]. It differs from simple averaging by prioritizing points with higher depth confidence, which helps mitigate noise and outliers in regions with uncertain geometry [S1, unofficial].

## How it works
**Inputs:**
- `points`: Tensor of shape $(N, 3)$ containing 3D coordinates of $N$ points [S1, unofficial].
- `voxel_indices`: Tensor of shape $(N)$ mapping each point to a voxel index [S1, unofficial].
- `depth_conf`: Optional tensor of shape $(N, 1)$ or $(N, D)$ containing depth confidence values for each point [S1, unofficial].
- `inverse_id`: Optional tensor of shape $(N)$ providing inverse mapping for unique voxel indices. If None, computed internally [S1, unofficial].
- `num_voxels`: Optional integer specifying the total number of voxels. If None, inferred from `inverse_id` [S1, unofficial].

**Outputs:**
- `centers`: Tensor of shape $(K, 3)$ containing the computed center for each of the $K$ active voxels [S1, unofficial].

**Data Structures:**
- `weighted_point_sums`: Tensor of shape $(num\_voxels, 3)$, accumulates $\sum (p_i \cdot w_i)$ [S1, unofficial].
- `weight_sums`: Tensor of shape $(num\_voxels, 1)$, accumulates $\sum w_i$ [S1, unofficial].
- `point_sums`: Tensor of shape $(num\_voxels, 3)$, accumulates $\sum p_i$ [S1, unofficial].
- `point_counts`: Tensor of shape $(num\_voxels, 1)$, accumulates $\sum 1$ [S1, unofficial].

**Steps:**
1. Validate input shape: Ensure `points` has shape $(N, 3)$. Raise `ValueError` if not [S1, unofficial].
2. Determine voxel indexing:
   - If `inverse_id` is None, compute it using `torch.unique(voxel_indices, return_inverse=True)` and set `num_voxels` to the number of unique indices [S1, unofficial].
   - If `inverse_id` is provided but `num_voxels` is None, set `num_voxels` to `inverse_id.max() + 1` [S1, unofficial].
3. Construct weights:
   - If `depth_conf` is None, create uniform weights $w_i = 1.0$ for all $i$ [S1, unofficial].
   - If `depth_conf` is provided, move it to the same device as `points`. If its shape is $(N, D)$, reshape to $(N, 1)$ by taking the mean over the last dimension [S1, unofficial]. Clamp minimum weight to 0 [S1, unofficial].
4. Initialize accumulators: Create `weighted_point_sums`, `weight_sums`, `point_sums`, and `point_counts` as zero-initialized tensors of appropriate shapes $(num\_voxels, 3)$ or $(num\_voxels, 1)$ [S1, unofficial].
5. Accumulate weighted points: `weighted_point_sums.index_add_(0, inverse_id, points * weights)` [S1, unofficial].
6. Accumulate weights: `weight_sums.index_add_(0, inverse_id, weights)` [S1, unofficial].
7. Accumulate raw points: `point_sums.index_add_(0, inverse_id, points)` [S1, unofficial].
8. Accumulate point counts: `point_counts.index_add_(0, inverse_id, torch.ones_like(weights))` [S1, unofficial].
9. Compute weighted centers: `weighted_centers = weighted_point_sums / weight_sums.clamp_min(1e-12)` [S1, unofficial].
10. Compute mean centers: `mean_centers = point_sums / point_counts.clamp_min(1)` [S1, unofficial].
11. Select final centers: For each voxel $j$, if `weight_sums[j] > 0`, use `weighted_centers[j]`; otherwise, use `mean_centers[j]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch

def compute_weighted_voxel_centers(points, voxel_indices, depth_conf=None, inverse_id=None, num_voxels=None):
    # Step 1: Validate input shape
    if points.shape[1] != 3:
        raise ValueError("Points must have shape (N, 3)")
    
    N = points.shape[0]
    
    # Step 2: Determine voxel indexing
    if inverse_id is None:
        unique_indices, inverse_id = torch.unique(voxel_indices, return_inverse=True)
        if num_voxels is None:
            num_voxels = unique_indices.shape[0]
    else:
        if num_voxels is None:
            num_voxels = inverse_id.max().item() + 1
            
    # Step 3: Construct weights
    if depth_conf is None:
        weights = torch.ones(N, 1, device=points.device, dtype=points.dtype)
    else:
        weights = depth_conf.to(points.device)
        if weights.dim() > 1 and weights.shape[1] > 1:
            weights = weights.mean(dim=1, keepdim=True)
        weights = torch.clamp(weights, min=0.0)
        
    # Step 4: Initialize accumulators
    weighted_point_sums = torch.zeros(num_voxels, 3, device=points.device, dtype=points.dtype)
    weight_sums = torch.zeros(num_voxels, 1, device=points.device, dtype=points.dtype)
    point_sums = torch.zeros(num_voxels, 3, device=points.device, dtype=points.dtype)
    point_counts = torch.zeros(num_voxels, 1, device=points.device, dtype=points.dtype)
    
    # Step 5: Accumulate weighted points
    weighted_point_sums.index_add_(0, inverse_id, points * weights)
    
    # Step 6: Accumulate weights
    weight_sums.index_add_(0, inverse_id, weights)
    
    # Step 7: Accumulate raw points
    point_sums.index_add_(0, inverse_id, points)
    
    # Step 8: Accumulate point counts
    point_counts.index_add_(0, inverse_id, torch.ones_like(weights))
    
    # Step 9: Compute weighted centers
    weighted_centers = weighted_point_sums / weight_sums.clamp_min(1e-12)
    
    # Step 10: Compute mean centers
    mean_centers = point_sums / point_counts.clamp_min(1)
    
    # Step 11: Select final centers
    # Use weighted centers where weights exist, else mean centers
    mask = (weight_sums > 0).expand_as(weighted_centers)
    centers = torch.where(mask, weighted_centers, mean_centers)
    
    return centers
```

**Parameters:**
- `clamp_min` for weights: 0.0 [S1, unofficial].
- `clamp_min` for division denominators: `1e-12` for weights, `1` for counts [S1, unofficial].

**Edge Cases and Pitfalls:**
- If `depth_conf` has multiple dimensions, it is averaged over the last dimension [S1, unofficial].
- Division by zero is prevented by clamping denominators [S1, unofficial].
- If a voxel has no points, `weight_sums` is 0, and `mean_centers` is used (though `point_counts` is also 0, so `clamp_min(1)` prevents division by zero, resulting in 0/1=0) [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
This method is part of the VGGT (Vision Geometry Grounded Transformer) pipeline, specifically within the `demo_colmap.py` script used for converting depth maps and poses into sparse voxel representations [S1, unofficial]. It leverages PyTorch's efficient tensor operations for accumulation [S1, unofficial].

## Sources
[S1] VGGT Demo Script, https://github.com/Zhaoyibinn/VGVGGT/blob/main/demo_colmap.py, fetched 2026-10-27 (unofficial)
