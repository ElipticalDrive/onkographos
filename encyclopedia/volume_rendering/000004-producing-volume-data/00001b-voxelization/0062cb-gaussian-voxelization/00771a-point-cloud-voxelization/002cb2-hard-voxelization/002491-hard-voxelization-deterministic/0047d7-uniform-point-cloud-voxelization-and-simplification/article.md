# Uniform Point Cloud Voxelization and Simplification
## Overview
A preprocessing technique to simplify large point clouds by voxelizing them and randomly sampling points within voxels to a target density, followed by re-projecting the simplified points onto multi-planes for aggregation weight calculation [S1, unofficial]. It sets itself apart from sibling hard voxelization methods by introducing a probabilistic downsampling step based on local point density, rather than deterministic retention or max-pooling, and immediately projects the result into a multi-plane representation for subsequent neural rendering [S1, unofficial].
## How it works
**Inputs:**
- Point cloud: $N$ points with 3D coordinates $(x, y, z)$ [S1, unofficial].
- Voxel parameters: `side_len` (voxel size in meters), `num_in_each_voxel` (target points per voxel) [S1, unofficial].
- Camera parameters: Intrinsic matrix $K$, Extrinsic matrix $E$ (camera-to-world), image dimensions $w \times h$ [S1, unofficial].
- Depth bounds: `valid_depth_near`, `valid_depth_far` [S1, unofficial].
- Multi-plane parameters: `num_planes` (default 32) [S1, unofficial].

**Outputs:**
- Simplified point indices and associated metadata for multi-plane aggregation [S1, unofficial].
- Data structures: `u_sorted` (uint16), `v_sorted` (uint16), `d_position` (uint8), `valid_position` (uint32), `groups_each` (uint32), `groups_index` (uint32), `groups_each_index` (uint16), `split_each_begin` (uint32), `split_each_begin_in_group` (uint32), `split_each_max` (uint16), `distance` (float16), `delta` (float32) [S1, unofficial].

**Data Structures:**
The output is organized into groups corresponding to voxels in the projected 2D space [S1, unofficial].
- `groups_each`: Array of size `num_planes`, storing the number of points in each plane's voxel grid [S1, unofficial].
- `groups_index`: Flattened list of point indices belonging to each group [S1, unofficial].
- `distance`: Depth of each point (float16) [S1, unofficial].
- `delta`: Weighting factor for aggregation (float32) [S1, unofficial].

**Steps:**

1.  **Bounding Box Calculation:**
    Compute the bounding box of the input point cloud:
    $$ \text{min}_x, \text{min}_y, \text{min}_z, \text{max}_x, \text{max}_y, \text{max}_z $$

2.  **Voxel Index Assignment:**
    For each point $i$ with coordinates $(x_i, y_i, z_i)$, compute the linear voxel index:
    $$ \text{num}_x = \lceil \frac{\text{max}_x - \text{min}_x}{\text{side\_len}} \rceil $$
    $$ \text{num}_y = \lceil \frac{\text{max}_y - \text{min}_y}{\text{side\_len}} \rceil $$
    $$ \text{idx}_i = \text{rint}\left(\frac{x_i - \text{min}_x}{\text{side\_len}}\right) + \text{rint}\left(\frac{y_i - \text{min}_y}{\text{side\_len}}\right) \cdot \text{num}_x + \text{rint}\left(\frac{z_i - \text{min}_z}{\text{side\_len}}\right) \cdot \text{num}_x \cdot \text{num}_y $$
    where $\text{rint}(\cdot)$ denotes rounding to the nearest integer [S1, unofficial].

3.  **Sorting:**
    Sort all points by their computed voxel index $\text{idx}_i$ to group points belonging to the same voxel together [S1, unofficial].

4.  **Probabilistic Sampling:**
    For each contiguous block of points with identical voxel indices:
    - Let $C$ be the count of points in the current voxel [S1, unofficial].
    - Let $T$ be the target number of points (`num_in_each_voxel`) [S1, unofficial].
    - Calculate selection probability for each point in the block:
      $$ p = \frac{T}{C} $$
    - Generate a random uniform number $r \in [0, 1)$ for each point [S1, unofficial].
    - Select the point if $p > r$ [S1, unofficial].

5.  **Re-projection to Multi-Planes:**
    - Calculate the transform matrix: $M = K \cdot E^{-1}$ [S1, unofficial].
    - Project 3D points to 2D image coordinates $(u, v)$:
      $$ u = \frac{x}{z}, \quad v = \frac{y}{z} $$
      where $z$ is the depth relative to the camera center [S1, unofficial].
    - Filter points based on valid image bounds ($0 \le u < w$, $0 \le v < h$) and depth bounds (`valid_depth_near` < depth < `valid_depth_far`) [S1, unofficial].

6.  **Depth Filtering:**
    - Sort valid points by depth in descending order [S1, unofficial].
    - Remove outliers: discard the 100 furthest and 50 nearest points [S1, unofficial].

7.  **Plane Assignment:**
    - Define plane boundaries by sampling random depths uniformly between `valid_d_max` and `valid_d_min` using `num_planes + 1` samples:
      $$ \text{boundaries} = \text{linspace}(\text{valid\_d\_max}, \text{valid\_d\_min}, \text{num\_planes} + 1) $$
    - Assign each point to a plane index $d_{\text{position}}$ based on which interval its depth falls into [S1, unofficial].

8.  **Grouping and Memory Allocation:**
    - Group points by their 2D grid position $(u, v)$ and plane index $d_{\text{position}}$ to form voxels in the projected space [S1, unofficial].
    - Calculate the maximum number of points in any single group/voxel for each plane to allocate memory buffers (`split_each_max`) [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def uniform_point_cloud_voxelization(points, side_len, num_in_each_voxel, 
                                     K, E, w, h, valid_depth_near, valid_depth_far, 
                                     num_planes=32):
    """
    points: (N, 3) array of (x, y, z)
    K: (3, 3) intrinsic matrix
    E: (4, 4) extrinsic matrix (camera to world)
    """
    # 1. Bounding Box
    min_xyz = np.min(points, axis=0)
    max_xyz = np.max(points, axis=0)
    min_x, min_y, min_z = min_xyz
    max_x, max_y, max_z = max_xyz
    
    # 2. Voxel Indices
    num_x = int(np.ceil((max_x - min_x) / side_len))
    num_y = int(np.ceil((max_y - min_y) / side_len))
    
    ix = np.round((points[:, 0] - min_x) / side_len).astype(int)
    iy = np.round((points[:, 1] - min_y) / side_len).astype(int)
    iz = np.round((points[:, 2] - min_z) / side_len).astype(int)
    
    voxel_indices = ix + iy * num_x + iz * num_x * num_y
    
    # 3. Sort by voxel index
    sort_order = np.argsort(voxel_indices)
    sorted_points = points[sort_order]
    sorted_indices = sort_order
    
    # 4. Probabilistic Sampling
    unique_indices, counts = np.unique(voxel_indices[sort_order], return_counts=True)
    
    selected_mask = np.zeros(len(points), dtype=bool)
    
    current_idx = 0
    for u_idx, count in zip(unique_indices, counts):
        end_idx = current_idx + count
        prob = num_in_each_voxel / count
        randoms = np.random.rand(count)
        selected_in_block = randoms < prob
        selected_mask[sorted_indices[current_idx:end_idx][selected_in_block]] = True
        current_idx = end_idx
        
    simplified_points = points[selected_mask]
    
    # 5. Re-projection
    E_inv = np.linalg.inv(E)
    homogeneous_points = np.hstack([simplified_points, np.ones((len(simplified_points), 1))])
    camera_points = (E_inv @ homogeneous_points.T).T[:, :3]
    
    x_c, y_c, z_c = camera_points[:, 0], camera_points[:, 1], camera_points[:, 2]
    
    u = x_c / z_c
    v = y_c / z_c
    
    valid_depth_mask = (z_c > valid_depth_near) & (z_c < valid_depth_far)
    valid_image_mask = (u >= 0) & (u < w) & (v >= 0) & (v < h)
    valid_mask = valid_depth_mask & valid_image_mask
    
    valid_points = simplified_points[valid_mask]
    valid_u = u[valid_mask]
    valid_v = v[valid_mask]
    valid_z = z_c[valid_mask]
    
    # 6. Depth Filtering
    depth_sort_order = np.argsort(-valid_z)
    sorted_valid_points = valid_points[depth_sort_order]
    sorted_valid_u = valid_u[depth_sort_order]
    sorted_valid_v = valid_v[depth_sort_order]
    sorted_valid_z = valid_z[depth_sort_order]
    
    if len(sorted_valid_points) > 150:
        final_points = sorted_valid_points[100:-50]
        final_u = sorted_valid_u[100:-50]
        final_v = sorted_valid_v[100:-50]
        final_z = sorted_valid_z[100:-50]
    else:
        final_points = sorted_valid_points
        final_u = sorted_valid_u
        final_v = sorted_valid_v
        final_z = sorted_valid_z
        
    # 7. Plane Assignment
    boundaries = np.linspace(valid_depth_far, valid_depth_near, num_planes + 1)
    d_positions = np.digitize(final_z, boundaries) - 1
    d_positions = np.clip(d_positions, 0, num_planes - 1)
    
    # 8. Grouping
    u_int = np.round(final_u).astype(int)
    v_int = np.round(final_v).astype(int)
    
    group_keys = (d_positions * (w * h)) + (v_int * w) + u_int
    
    group_sort_order = np.argsort(group_keys)
    sorted_group_keys = group_keys[group_sort_order]
    sorted_final_points = final_points[group_sort_order]
    sorted_final_z = final_z[group_sort_order]
    sorted_final_u = final_u[group_sort_order]
    sorted_final_v = final_v[group_sort_order]
    
    unique_groups, group_counts = np.unique(sorted_group_keys, return_counts=True)
    
    groups_each = np.zeros(num_planes, dtype=np.uint32)
    for g, c in zip(unique_groups, group_counts):
        plane = g // (w * h)
        groups_each[plane] += c
        
    groups_index = np.arange(len(sorted_group_keys), dtype=np.uint32)
    
    split_each_max = np.zeros(num_planes, dtype=np.uint16)
    for g, c in zip(unique_groups, group_counts):
        plane = g // (w * h)
        if c > split_each_max[plane]:
            split_each_max[plane] = c
            
    # Compute split indices for output layout
    split_each_begin = np.zeros(num_planes, dtype=np.uint32)
    current_offset = 0
    for plane in range(num_planes):
        split_each_begin[plane] = current_offset
        current_offset += groups_each[plane]
        
    split_each_begin_in_group = np.zeros(len(unique_groups), dtype=np.uint32)
    current_offset = 0
    for i, c in enumerate(group_counts):
        split_each_begin_in_group[i] = current_offset
        current_offset += c
        
    groups_each_index = np.zeros(len(sorted_group_keys), dtype=np.uint32)
    current_offset = 0
    for i, c in enumerate(group_counts):
        groups_each_index[current_offset:current_offset + c] = np.arange(c, dtype=np.uint32)
        current_offset += c
        
    # delta: weighting factor for aggregation. Not given in the sources.
    delta = np.zeros(len(sorted_group_keys), dtype=np.float32)
    
    return {
        'u_sorted': sorted_final_u.astype(np.uint16),
        'v_sorted': sorted_final_v.astype(np.uint16),
        'd_position': d_positions[group_sort_order].astype(np.uint8),
        'valid_position': np.arange(len(sorted_group_keys), dtype=np.uint32),
        'groups_each': groups_each,
        'groups_index': groups_index,
        'groups_each_index': groups_each_index,
        'split_each_begin': split_each_begin,
        'split_each_begin_in_group': split_each_begin_in_group,
        'split_each_max': split_each_max,
        'distance': sorted_final_z.astype(np.float16),
        'delta': delta
    }
```

**Parameters:**
- `side_len`: Voxel side length in meters. Typical values: 0.01 or 0.02 [S1, unofficial].
- `num_in_each_voxel`: Predefined number of points to retain in each voxel after simplification. Typical values: 30 or 20 [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Empty Voxels:** Voxels with no points are ignored [S1, unofficial].
- **Precision:** Depth filtering relies on `float16` for distance storage, which may introduce quantization errors for very large scenes [S1, unofficial].
- **Boundary Conditions:** Points exactly on the bounding box edges or depth boundaries must be handled consistently with the rounding and clipping logic [G].
- **Outlier Removal:** If the total number of valid points is less than 150, the outlier removal step may discard all points. A guard should check the length before slicing [G].

## Operating characteristics
- **Time Complexity:** Dominated by sorting points by voxel index ($O(N \log N)$) and sorting by depth ($O(N \log N)$) [S1, unofficial]. Probabilistic sampling is $O(N)$ [S1, unofficial].
- **Memory Complexity:** $O(N)$ for storing point data and intermediate arrays [S1, unofficial]. Output size depends on the number of non-empty voxels and points retained [S1, unofficial].
- **Measured Performance:** Not given in the sources [S1, unofficial].
- **Strengths:** Reduces point cloud density significantly while preserving structure; probabilistic sampling avoids bias towards specific points; multi-plane projection facilitates neural rendering [S1, unofficial].
- **Limitations:** Stochastic nature means results vary between runs; outlier removal may discard valid data if the cloud is sparse; requires careful tuning of `side_len` and `num_in_each_voxel` [S1, unofficial].
- **When to Use:** Preprocessing for neural radiance fields (NeRF) or similar neural rendering techniques where point cloud density is too high for direct processing [S1, unofficial].
- **When Not to Use:** When deterministic voxelization is required; when exact point preservation is necessary [G].

## Background
Originated in the context of neural rendering and point cloud processing [S1, unofficial]. It was developed to handle large-scale point clouds by simplifying them before projection into multi-plane representations for efficient neural network training [S1, unofficial].

## Sources
[S1] https://github.com/Mayxmu/PCMP-NeRF, https://github.com/Mayxmu/PCMP_NeRF, fetched 2026-10-30 (unofficial)
