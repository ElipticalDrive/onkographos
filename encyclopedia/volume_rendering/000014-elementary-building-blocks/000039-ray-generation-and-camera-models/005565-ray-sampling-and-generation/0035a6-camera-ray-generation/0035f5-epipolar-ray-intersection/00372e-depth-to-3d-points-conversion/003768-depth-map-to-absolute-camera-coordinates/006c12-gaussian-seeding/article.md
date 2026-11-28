# Gaussian Seeding

Also called: Gaussian Seeding and Sampling, New Gaussian Seeding

## Overview
Gaussian Seeding initializes the 3D Gaussian primitives for a SLAM or reconstruction system by sampling points from a ground truth depth map and camera pose. It solves the problem of establishing an initial point cloud representation that balances uniform spatial coverage with high-frequency detail preservation. It differs from simple uniform sampling by incorporating gradient-based pixel selection and handles submap initialization logic to manage memory constraints during incremental mapping.

## How it works
**Inputs:**
- `gt_color`: Ground truth color image.
- `gt_depth`: Ground truth depth map.
- `intrinsics`: Camera intrinsic matrix.
- `c2w`: Estimated camera-to-world transformation matrix.
- `seeding_mask`: Boolean mask indicating valid pixels for seeding.
- `is_new_submap`: Boolean flag indicating if a new submap is being initialized.
- `new_submap_points_num`: Target number of points for a new submap.
- `new_submap_gradient_points_num`: Number of gradient-based points for a new submap.
- `new_frame_sample_size`: Target number of points for a new frame (if not new submap).

**Outputs:**
- `points`: A tensor of 3D coordinates in world space, cast to `float32`.

**Data Structures:**
- `pts`: 3D point cloud tensor of shape $(N, 3)$, where $N$ is the number of valid pixels.
- `valid_ids`: 1D tensor of flat indices corresponding to pixels where `seeding_mask` is true and depth is valid.
- `sample_ids`: 1D tensor of selected indices to generate the final Gaussian set.

**Steps:**

1.  **Point Cloud Creation**: Convert the ground truth depth map to 3D points in world coordinates. The depth is scaled by a factor of $1.005$ to account for sensor bias or offset.
    $$ P_i = \text{depth\_to\_world}(gt\_color_i, 1.005 \times gt\_depth_i, intrinsics, c2w) $$
    Let $pts$ be the resulting tensor of shape $(H \times W, 3)$.

2.  **Masking**: Flatten the ground truth depth. Create a boolean mask `non_zero_depth_mask` where $depth > 0$. This filters out invalid depth values (e.g., infinity or zero).

3.  **Valid ID Identification**: Identify `valid_ids` as the flat indices of elements in `seeding_mask` that are true. These are the candidate pixels for seeding.

4.  **Sampling Strategy Selection**:
    -   **Case A: New Submap (`is_new_submap` is True)**
        1.  **Uniform Sampling**: If `new_submap_points_num` < 0, select all indices from `pts.shape[0]`. Otherwise, sample `new_submap_points_num` unique indices from the total pixel count without replacement. Let this set be `uniform_ids`.
        2.  **Gradient Sampling**: Sample `new_submap_gradient_points_num` pixel indices based on the image gradient magnitude of `gt_color`. Let this set be `gradient_ids`.
        3.  **Combination**: Concatenate `uniform_ids` and `gradient_ids`. Combine with `valid_ids` and take the unique values to form the final `sample_ids`.
    -   **Case B: New Frame / Existing Submap (`is_new_submap` is False)**
        1.  If `new_frame_sample_size` < 0 or the number of `valid_ids` is less than `new_frame_sample_size`, use all `valid_ids`.
        2.  Otherwise, randomly sample `new_frame_sample_size` unique indices from `valid_ids` without replacement.
        3.  Set `sample_ids` to these selected indices.

5.  **Filtering**: Filter `sample_ids` to ensure they correspond only to pixels where `non_zero_depth_mask` is true. This ensures no NaN or invalid coordinates are included.

6.  **Output**: Return the 3D points $pts[sample\_ids]$, cast to `float32`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_seeding(gt_color, gt_depth, intrinsics, c2w, seeding_mask, 
                     is_new_submap, new_submap_points_num, 
                     new_submap_gradient_points_num, new_frame_sample_size):
    # Step 1: Create point cloud from ground truth
    # Depth is scaled by 1.005
    pts = create_point_cloud(gt_color, 1.005 * gt_depth, intrinsics, c2w)
    
    # Step 2: Flatten depth and create mask for valid depths
    flat_gt_depth = gt_depth.flatten()
    non_zero_depth_mask = flat_gt_depth > 0
    
    # Step 3: Get valid IDs from seeding mask
    # Assuming seeding_mask is flattened boolean tensor
    valid_ids = torch.nonzero(seeding_mask.flatten(), as_tuple=False).squeeze()
    
    sample_ids = None
    
    if is_new_submap:
        # Step 4A: New Submap Logic
        # Uniform sampling
        if new_submap_points_num < 0:
            uniform_ids = torch.arange(pts.shape[0])
        else:
            uniform_ids = torch.randperm(pts.shape[0])[:new_submap_points_num]
            
        # Gradient sampling
        gradient_ids = sample_pixels_based_on_gradient(gt_color, new_submap_gradient_points_num)
        
        # Combine and unique
        combined_ids = torch.cat([uniform_ids, gradient_ids, valid_ids])
        sample_ids = torch.unique(combined_ids)
    else:
        # Step 4B: New Frame / Existing Submap Logic
        num_valid = valid_ids.shape[0]
        if new_frame_sample_size < 0 or num_valid < new_frame_sample_size:
            sample_ids = valid_ids
        else:
            # Sample from valid_ids without replacement
            perm = torch.randperm(num_valid)
            sample_ids = valid_ids[perm[:new_frame_sample_size]]
            
    # Step 5: Filter by non-zero depth mask
    # Ensure sample_ids are within bounds and have valid depth
    valid_depth_mask = non_zero_depth_mask[sample_ids]
    final_ids = sample_ids[valid_depth_mask]
    
    # Step 6: Return points
    return pts[final_ids].to(torch.float32)
```

**Parameters:**
- `1.005`: Depth scaling factor [S1, unofficial].
- `new_submap_points_num`: Controls uniform sampling density for new submaps. Negative values imply "all points" [S1, unofficial].
- `new_submap_gradient_points_num`: Number of high-detail points sampled via gradient [S1, unofficial].
- `new_frame_sample_size`: Controls sampling density for incremental frames [S1, unofficial].

**Edge Cases:**
- If `valid_ids` is empty, the output is an empty tensor.
- If `sample_ids` contains indices where depth is zero or invalid, they are removed in Step 5.
- Precision: Output is explicitly cast to `float32` [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Dominated by the point cloud creation ($O(H \times W)$) and sorting/unique operations on indices ($O(N \log N)$ where $N$ is the number of selected pixels). Gradient sampling adds overhead proportional to image size.
- **Memory Complexity**: $O(H \times W)$ for storing the full point cloud before sampling.
- **Strengths**: Combines global coverage (uniform) with local detail (gradient), ensuring robust initialization for SLAM systems.
- **Limitations**: Relies on accurate ground truth depth and camera pose. If `c2w` is poor, the seeded Gaussians will be misaligned.

## Background
Gaussian Seeding is a method used in Gaussian Splatting-based SLAM systems, specifically referenced in the context of the Gaussian-SLAM project. It serves as the initialization step for creating the 3D Gaussian representation from monocular or stereo depth inputs.

## Sources
[S1] VladimirYugay/Gaussian-SLAM, https://github.com/VladimirYugay/Gaussian-SLAM, fetched 2026-10-27 (unofficial)
