# Point Cloud Density Evaluation


## Overview
Point Cloud Density Evaluation updates the intensity values of a set of 3D points based on a volumetric field, effectively sampling the volume at the point locations. It solves the problem of transferring scalar field data from a regular grid representation to a non-regular point cloud representation, which is often required for density-based regularization or supervision in neural rendering pipelines. It differs from its siblings by focusing on the direct interpolation and optional footprint pooling of intensities rather than structural mesh generation or adaptive subdivision.

## How it works
The method takes a volumetric tensor and a set of 3D points, interpolates the volume values at the point coordinates, and optionally aggregates values from neighboring voxels (footprint pooling) before normalizing the result.

**Inputs:**
- `volume`: A 5D tensor of shape `[1, C, D, H, W]` representing the volume, where `C` is channels, `D` is depth, `H` is height, and `W` is width. The values are assumed to be in the working data type (typically `torch.float32`).
- `points`: A tensor of point coordinates with shape `[N, 3]`. Coordinates are in normalized space `[0, 1]`.
- `scale`: Optional tensor of scale parameters for each point, shape `[N, 3]` or `[N]`. Used to determine the footprint size for pooling.
- `min_val`: Optional scalar for the minimum volume value. If `None`, computed from `volume`.
- `max_val`: Optional scalar for the maximum volume value. If `None`, computed from `volume`.
- `normalize`: Boolean flag. If `True`, intensities are normalized to `[0, 1]`. Default `False`.
- `enable_footprint_pooling`: Boolean flag. If `True`, aggregates values from a local voxel neighborhood around each point. Default `False` in the primary call, but logic exists to handle it.

**Outputs:**
- `intensities`: Tensor of shape `[N]` containing the sampled and processed intensity values.
- `volume_min`: The minimum value used for normalization.
- `volume_max`: The maximum value used for normalization.

**Data Structures:**
- `volume_f`: The volume tensor cast to `work_dtype` (float32).
- `grid_large`: A grid of coordinates for `F.grid_sample`, derived from `points`.

**Steps:**
1. **Empty Check:** If `points.numel() == 0`, return an empty tensor for intensities, and `volume.min()`, `volume.max()` as min/max values.
2. **Type Conversion:** Convert `volume` to `work_dtype` (float32).
3. **Volume Bounds:** Determine `volume_min` and `volume_max`. If `min_val`/`max_val` are provided, use them; otherwise, compute `volume_f.min()` and `volume_f.max()`.
4. **Base Sampling:** Call `sample_intensities_from_volume` with `enable_footprint_pooling=False` to obtain `base_samples` (raw interpolated values). This uses bilinear interpolation via `F.grid_sample` with `align_corners=True`.
   - The grid coordinates are derived from the points.
   - Formula for sampling:
     $$ \text{samples} = \text{F.grid\_sample}(\text{volume}, \text{grid}, \text{mode}=\text{"bilinear"}, \text{padding\_mode}=\text{padding\_mode}, \text{align\_corners}=\text{True}) $$
   - The output is reshaped to `[idx.shape[0], -1]`.
5. **Footprint Pooling (Conditional):** If `enable_footprint_pooling` is `True` and `scale` is valid:
   - Compute coverage mask indices.
   - Convert points to voxel coordinates.
   - Compute voxel extent:
     $$ \text{extent\_xyz} = \frac{\text{scale\_cov}}{\text{voxel\_size\_xyz}.unsqueeze(0)} $$
     Reorder to `extent_zyx`.
   - Compute bounding box indices in voxel space:
     $$ \text{min\_idx} = \lfloor \text{point\_voxel} - \text{extent} \rfloor $$
     $$ \text{max\_idx} = \lceil \text{point\_voxel} + \text{extent} \rceil $$
     Clamp `min_idx` and `max_idx` to volume dimensions `[0, D-1]`, `[0, H-1]`, `[0, W-1]`.
   - Iterate over each point:
     - Extract voxel region `[z0:z1+1, y0:y1+1, x0:x1+1]`.
     - Compute the mean of the values in this region.
     - Update the sample value with this mean.
6. **Normalization (Conditional):** If `normalize` is `True`:
   - Compute denominator:
     $$ \text{denominator} = \max(\text{volume\_max} - \text{volume\_min}, 10^{-8}) $$
   - Normalize intensities:
     $$ \text{intensities} = \frac{\text{intensities} - \text{volume\_min}}{\text{denominator}} $$
   - Clamp `intensities` to `[0, 1]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def point_cloud_density_evaluation(volume, points, scale=None, min_val=None, max_val=None, normalize=False, enable_footprint_pooling=False):
    """
    Updates intensity values for points based on their current positions.
    
    Args:
        volume: Tensor [1, C, D, H, W]
        points: Tensor [N, 3] in [0, 1] space
        scale: Optional Tensor [N, 3] or [N]
        min_val: Optional float
        max_val: Optional float
        normalize: bool
        enable_footprint_pooling: bool
        
    Returns:
        intensities: Tensor [N]
        volume_min: float
        volume_max: float
    """
    # Handle empty points
    if points.numel() == 0:
        return torch.empty(0), volume.min(), volume.max()

    # Convert to work dtype
    volume_f = volume.to(torch.float32)
    
    # Determine volume bounds
    if min_val is None:
        volume_min = volume_f.min()
    else:
        volume_min = min_val
        
    if max_val is None:
        volume_max = volume_f.max()
    else:
        volume_max = max_val

    # Base sampling without footprint pooling
    # Note: In a real implementation, grid_large is constructed from points
    # For this pseudocode, we assume a helper `construct_grid` exists or is implicit
    # The source code snippet shows:
    # samples_large = F.grid_sample(volume_5d, grid_large, mode="bilinear", padding_mode=padding_mode, align_corners=True).view(idx.shape[0], -1)
    
    # Assuming points are in [N, 3] and volume is [1, C, D, H, W]
    # We need to create a grid for grid_sample. 
    # Standard grid_sample expects grid of shape [N, H_out, W_out, 2] for 2D or [N, D_out, H_out, W_out, 3] for 3D.
    # Here, we assume a simplified 1D sampling per point for intensity (C=1 or reduced).
    # The source implies a direct mapping.
    
    # Construct grid for 3D grid_sample
    # points are [N, 3] in [0, 1]. grid_sample expects coordinates in [-1, 1] if align_corners=False, 
    # but align_corners=True maps [0, 1] to [0, 1] relative to corners.
    # However, PyTorch grid_sample typically expects normalized coordinates.
    # Let's assume the implementation handles the coordinate transformation internally or via `grid_large`.
    
    # For the sake of self-contained pseudocode, we simulate the grid creation:
    # grid_large shape: [1, N, 1, 1, 3] for 3D volume sampling
    grid_large = points.unsqueeze(0).unsqueeze(2).unsqueeze(3) # [1, N, 1, 1, 3]
    
    # Perform sampling
    # Note: F.grid_sample on 5D input (N, C, D, H, W) with 5D grid (N, D, H, W, 3)
    # The source uses `volume_5d` and `grid_large`.
    # We assume C=1 for intensity. If C>1, this needs adjustment.
    # The source snippet: `.view(idx.shape[0], -1)` suggests output is [N, C].
    
    samples_large = F.grid_sample(
        volume_f,
        grid_large,
        mode="bilinear",
        padding_mode="zeros", # Assumed default if not specified, source says `padding_mode`
        align_corners=True
    )
    
    # Reshape to [N, C] -> take first channel if C>1, or just [N] if C=1
    # The source says `view(idx.shape[0], -1)`. Let's assume we want the intensity (first channel).
    if samples_large.shape[1] > 1:
        intensities = samples_large[:, 0]
    else:
        intensities = samples_large.squeeze(1)

    # Footprint pooling logic
    if enable_footprint_pooling and scale is not None:
        # Convert points to voxel coordinates
        # Assuming volume dimensions D, H, W
        D, H, W = volume_f.shape[2], volume_f.shape[3], volume_f.shape[4]
        
        # voxel coordinates
        voxel_coords = points * torch.tensor([D, H, W], device=points.device).float()
        
        # Compute scale_cov (assuming scale is provided as radius or similar)
        # Source: extent_xyz = scale_cov / voxel_size_xyz.unsqueeze(0)
        # voxel_size_xyz is typically 1.0 if coordinates are in voxel indices
        # Let's assume scale_cov is derived from `scale`
        if scale.dim() == 1:
            scale_cov = scale.unsqueeze(1) # [N, 1]
        else:
            scale_cov = scale # [N, 3]
            
        # voxel_size_xyz is 1.0 in voxel index space
        voxel_size_xyz = torch.tensor([1.0, 1.0, 1.0], device=points.device)
        
        extent_xyz = scale_cov / voxel_size_xyz.unsqueeze(0)
        # Reorder to zyx
        extent_zyx = extent_xyz[:, [2, 1, 0]] # Assuming scale_cov was xyz? Source says scale_cov / voxel_size_xyz.
        # Actually, source says `extent_xyz = scale_cov / voxel_size_xyz.unsqueeze(0)`, then `reorder to extent_zyx`.
        # If scale_cov is [N, 3] (xyz), then extent_xyz is [N, 3] (xyz).
        # Reorder to zyx: [z, y, x]
        extent_zyx = extent_xyz[:, [2, 1, 0]]
        
        # Compute min_idx and max_idx
        # floor/ceil of voxel_coords +/- extent
        min_idx = torch.floor(voxel_coords - extent_zyx).long()
        max_idx = torch.ceil(voxel_coords + extent_zyx).long()
        
        # Clamp to volume dimensions
        min_idx = torch.clamp(min_idx, 0, torch.tensor([D-1, H-1, W-1], device=points.device))
        max_idx = torch.clamp(max_idx, 0, torch.tensor([D-1, H-1, W-1], device=points.device))
        
        # Iterate and compute mean
        new_intensities = torch.zeros_like(intensities)
        for i in range(len(points)):
            z0, y0, x0 = min_idx[i]
            z1, y1, x1 = max_idx[i]
            
            # Extract region
            # Note: This is a naive loop. In practice, this would be vectorized.
            region = volume_f[0, 0, z0:z1+1, y0:y1+1, x0:x1+1]
            if region.numel() > 0:
                new_intensities[i] = region.mean()
            else:
                new_intensities[i] = intensities[i] # Fallback
        
        intensities = new_intensities

    # Normalization
    if normalize:
        denominator = max(volume_max - volume_min, 1e-8)
        intensities = (intensities - volume_min) / denominator
        intensities = torch.clamp(intensities, 0.0, 1.0)

    return intensities, volume_min, volume_max
```

**Parameters:**
- `points`: Point coordinates in normalized `[0, 1]` space, shape `[N, 3]`.
- `scale`: Optional scale parameters for each point, shape `[N, 3]` or `[N]`.
- `normalize`: Whether to normalize intensities to `[0, 1]` range (default `False`).

**Edge Cases and Pitfalls:**
- **Empty Points:** If the input point cloud is empty, the function returns an empty tensor and the volume's min/max.
- **Precision:** Use `float32` for volume conversion to avoid precision issues during interpolation.
- **Boundaries:** Voxel indices must be clamped to `[0, D-1]`, `[0, H-1]`, `[0, W-1]` to prevent out-of-bounds access.
- **Denominator Zero:** The normalization denominator uses `max(..., 1e-8)` to prevent division by zero if `volume_min == volume_max`.

## Subtopics

### [Chamfer Distance](2446-chamfer-distance.md)
Calculates a symmetric distance between two point clouds.

### [Neighbor Counting](2905-neighbor-counting.md)

### [BrickDensityRegion Point Splatting](3956-brickdensityregion-point-splatting.md)

### [Mesh/Point Cloud Evaluation Metrics](4797-mesh-point-cloud-evaluation-metrics.md)
Computes comprehensive mesh quality metrics including Chamfer Distance (CD), Normal Consistency (NC), F1 score, Edge Chamfer Distance (ECD), Edge F1 (EF1), aspect ratio, radius ratio, angle regularity, and self-intersection percentage.

### [Probabilistic Occupancy Policy](5043-probabilistic-occupancy-policy.md)
A policy for converting point cloud intensity values into occupancy status (FREE, OCCUPIED, UNKNOWN) based on proximity to 0.0 (FREE) or 1.0 (OCCUPIED) within a tolerance range.

### Poisson Distribution Lookup for Feature Points
Given a uniformly distributed random number, returns the number of feature points in a given cube based on Poisson distribution thresholds. If the value is less than 0.0915781944272058, return 1. If less than 0.238103305510735, return 2. If less than 0.433470120288774, return 3. If less than 0.628836935299644, return 4. If less than 0.785130387122075, return 5. If less than 0.889326021747972, return 6. If less than 0.948866384324819, return 7. If less than 0.978636565613243, return 8. [S2, unofficial]

## Operating characteristics
- **Time Complexity:** $O(N)$ for base sampling, where $N$ is the number of points. Footprint pooling adds complexity proportional to the number of points times the volume of the footprint region.
- **Memory Complexity:** $O(N)$ for storing intensities.
- **Performance:** The method is efficient for dense point clouds due to vectorized grid sampling. Footprint pooling can be slower if not optimized.
- **Strengths:** Direct interpolation, flexible footprint pooling, easy integration with PyTorch.
- **Limitations:** Requires regular volume grid. Footprint pooling implementation in pseudocode is naive (loop-based).

## Background
This method is part of the 3D Gaussian Splatting acceleration workflow, specifically for volume-based supervision. It builds on the original 3D Gaussian Splatting work by Kerbl et al. [S1, unofficial].

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S2] Scrawk/Marching-Cubes, https://github.com/Scrawk/Marching-Cubes, fetched 2026-10-26 (unofficial)
