# Error-Driven Point Resampling & Grid Aggregation


## Overview
This method computes a 3D error density field by accumulating per-pixel rendering discrepancies across multiple camera views into a voxel grid, then uses this density to guide the resampling of points for tetrahedral grid refinement. It distinguishes itself from its parent [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md) by driving the resampling probability based on quantitative rendering error rather than topological neighbor status.

## How it works
**Inputs:**
- A set of input points (typically from a previous resampling stage or initial sampling).
- A list of camera views (position, orientation, intrinsics).
- A target voxel grid resolution (`voxel_resolution`).
- Camera image resolution (`resolution`).

**Outputs:**
- A refined set of points distributed according to the computed error density.
- A voxel grid containing mean error values per voxel.

**Data Structures:**
- `Grid`: A 3D voxel grid structure.
  - Fields: `density` (float tensor, shape `[voxel_resolution, voxel_resolution, voxel_resolution]`), `counts` (int tensor, same shape), `voxel_resolution` (int).
  - Initialization: `Grid(voxel_resolution, -1.0, 2.0)` [S1, unofficial]. The bounds `-1.0` and `2.0` likely define the normalized coordinate space or scaling factors for the grid, though exact semantic meaning is not explicitly detailed beyond initialization [S1, unofficial].
- `voxel_indices`: 1D integer tensor mapping 3D coordinates to flattened grid indices.

**Steps:**
1. **Initialize Grid:** Create a 3D grid with resolution `voxel_resolution`. Initialize error accumulation tensors (`grid_point_errors`) and count tensors (`grid_point_counts`) to zero. The grid bounds are set to `-1.0` and `2.0` [S1, unofficial].
2. **Iterate Cameras:** For each camera in the view list:
   a. **Render:** Render the current point cloud to obtain position masks (`current_mask`) and depth/position information using `render_pos_mask` [S1, unofficial].
   b. **Filter Pixels:** Identify valid pixels where `current_mask.squeeze(-1) > 0.5` [S1, unofficial].
   c. **Compute Error:** Calculate the per-pixel rendering error for these valid pixels. The specific error metric (e.g., L1, L2, depth difference) is not explicitly defined in the source, but it is aggregated into `masked_error` [S1, unofficial].
   d. **Map to Voxel Indices:** Convert the 3D positions of the filtered pixels to voxel indices using `grid.point_voxel_idx` [S1, unofficial].
   e. **Validate Indices:** Ensure the computed voxel indices are within the valid bounds `[0, voxel_resolution-1]` in all dimensions [S1, unofficial].
   f. **Flatten and Scatter:** Flatten the valid voxel indices into a 1D tensor. Scatter-add the `masked_error` values to `grid_point_errors` and increment `grid_point_counts` at the corresponding flattened indices [S1, unofficial].
3. **Accumulate Across Cameras:** Repeat steps 2a-2f for all cameras. The `grid_point_errors` and `grid_point_counts` tensors accumulate values across all views [S1, unofficial].
4. **Compute Mean Error:** Calculate the mean error per voxel:
   $$ \text{grid\_point\_errors} = \begin{cases} \frac{\text{grid\_point\_errors}}{\text{grid\_point\_counts}} & \text{if } \text{grid\_point\_counts} > 0 \\ \text{grid\_point\_errors} & \text{otherwise} \end{cases} $$
   This is implemented using `torch.where` [S1, unofficial].
5. **Assign Density:** Assign the computed mean error values to `grid.density` [S1, unofficial].
6. **Resample Points:** Use `grid.sample_points` to generate a new set of points. The sampling strategy is controlled by `resampled_points_method` (default 'partial') [S1, unofficial]. The probability of sampling a point in a voxel is proportional to its `density` value [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def error_driven_resampling(points, cameras, voxel_resolution=32, resolution=[1024, 1024], aggregation_method='interpolation', resampled_points_method='partial', device='cuda'):
    # Initialize grid structures
    grid = Grid(voxel_resolution, -1.0, 2.0)
    grid_point_errors = torch.zeros(voxel_resolution**3, device=device)
    grid_point_counts = torch.zeros(voxel_resolution**3, dtype=torch.int, device=device)
    
    for camera in cameras:
        # Render positions and masks
        positions, current_mask = render_pos_mask(points, camera, resolution)
        
        # Filter valid pixels
        valid_mask = current_mask.squeeze(-1) > 0.5
        valid_positions = positions[valid_mask]
        valid_errors = compute_error(points, camera, valid_positions) # Placeholder for specific error metric
        
        # Convert to voxel indices
        voxel_indices = grid.point_voxel_idx(valid_positions)
        
        # Validate indices
        valid_indices = (voxel_indices >= 0).all(dim=1) & (voxel_indices < voxel_resolution).all(dim=1)
        valid_indices = valid_indices.nonzero().squeeze()
        
        if len(valid_indices) == 0:
            continue
            
        flattened_indices = voxel_indices[valid_indices].flatten()
        masked_errors = valid_errors[valid_indices]
        
        # Scatter add errors and counts
        grid_point_errors.scatter_add_(0, flattened_indices, masked_errors)
        grid_point_counts.scatter_add_(0, flattened_indices, torch.ones_like(flattened_indices, dtype=torch.int))
        
    # Compute mean error
    mean_errors = torch.where(grid_point_counts > 0, grid_point_errors / grid_point_counts.float(), grid_point_errors)
    
    # Assign to grid density
    grid.density = mean_errors.reshape(voxel_resolution, voxel_resolution, voxel_resolution)
    
    # Resample points
    new_points = grid.sample_points(resampled_points_method)
    
    return new_points
```

**Parameters:**
- `voxel_resolution`: int, default 32. Grid resolution in each dimension [S1, unofficial].
- `resolution`: list, default [1024, 1024]. Camera image resolution [S1, unofficial].
- `aggregation_method`: str, default 'interpolation'. Method for aggregating sampled points (contextual, likely used in `render_pos_mask` or error calculation) [S1, unofficial].
- `resampled_points_method`: str, default 'partial'. Strategy for point resampling (e.g., partial, percentage, full) [S1, unofficial].
- `device`: str, default 'cuda'. Compute device [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Counts:** Voxels with no valid pixel projections have zero counts. The mean error calculation must handle division by zero, typically by leaving the error at zero or a default value [S1, unofficial].
- **Boundary Validation:** Voxel indices must be strictly validated against grid bounds to prevent out-of-bounds memory access during scatter operations [S1, unofficial].
- **Precision:** Accumulating errors across many cameras may lead to floating-point precision issues if not managed carefully, though standard float32 is implied by PyTorch usage [S1, unofficial].

## Subtopics

### Point Resampling with Neighbor Masking
Resamples points based on active/neighbor status, supporting partial, percentage, and full resampling strategies. This is the parent method, which this method extends by using error density instead of neighbor topology. [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md)

## Operating characteristics
- **Time Complexity:** $O(N \cdot V)$, where $N$ is the number of points and $V$ is the number of cameras. Each camera requires rendering and scattering operations proportional to the number of projected points.
- **Memory Complexity:** $O(R^3)$, where $R$ is the `voxel_resolution`. The grid size dominates memory usage.
- **Strengths:** Directly targets areas of high rendering error, leading to more efficient point distribution for visual quality.
- **Limitations:** Requires multiple camera views and rendering passes. Computationally expensive for high-resolution grids or many cameras.
- **When to Use:** When visual fidelity is critical and computational resources allow for multi-view error accumulation.
- **When Not to Use:** When real-time performance is required or only a single view is available.

## Background
This method is part of the TetWeave project, which focuses on tetrahedral grid refinement for volume rendering. It was developed by Alexandre Binninger. The method is designed to work in conjunction with blue noise sampling and neighbor masking strategies to produce high-quality volumetric data.

## Sources
[S1] AlexandreBinninger/TetWeave, https://github.com/AlexandreBinninger/TetWeave, fetched 2026-10-27 (unofficial)
