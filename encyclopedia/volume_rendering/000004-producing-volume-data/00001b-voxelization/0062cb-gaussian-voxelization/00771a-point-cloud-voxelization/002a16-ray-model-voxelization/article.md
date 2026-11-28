# Ray Model Voxelization

Also called: LiDAR Voxelization, Ray Traversal Voxelization

## Overview
Ray Model Voxelization converts sparse LiDAR point clouds into dense 3D occupancy grids by tracing rays from sensor origins to detected points, classifying voxels as free, occupied, or unobserved [S1]. It differs from [Hard Voxelization](5345-hard-voxelization.md) by explicitly modeling the empty space between the sensor and the target points, rather than only marking voxels containing points, and from [Point Cloud to Gaussian Converter](1788-point-cloud-to-gaussian-converter.md) by producing discrete grid states rather than continuous density fields [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   `points_origin`: Sensor origin positions, shape $[N, 3]$ [S1].
    *   `points`: LiDAR point coordinates, shape $[N, 3+]$ (typically $[N, 3]$ or $[N, 4]$ with intensity) [S1].
    *   `points_label`: Semantic labels for points, shape $[N]$ [S1].
    *   `point_cloud_range`: 3D bounding box $[x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$ in meters [S1].
    *   `voxel_size`: Voxel dimensions $[v_x, v_y, v_z]$ in meters. Default: $[0.2, 0.2, 0.2]$ [S1].
    *   `spatial_shape`: Output grid dimensions $[H, W, D]$. Default: $[512, 256, 32]$ [S1].
*   **Outputs:**
    *   `voxel_state`: Integer grid $[H, W, D]$ with values $-1$ (NOT_OBSERVED), $0$ (FREE), $1$ (OCCUPIED) [S1].
    *   `voxel_label`: Integer grid $[H, W, D]$ with semantic labels. Default label for free voxels is $0$ [S1].
    *   `invalid_mask`: Boolean grid $[H, W, D]$ indicating unobserved voxels or voxels with no label [S1].

**Data Structures**
The implementation uses CUDA kernels for parallel ray traversal [S1]. Key variables include:
*   `voxel_occ_count`: Atomic counter for occupied voxels [S1].
*   `voxel_free_count`: Atomic counter for free voxels [S1].
*   `ray[3]`: Ray direction vector [S1].
*   `step[3]`: Step direction per axis ($+1$ or $-1$) [S1].
*   `tDelta[3]`: Increment in $t$ to move one voxel [S1].
*   `tMax[3]`: Next voxel boundary in $t$ units [S1].
*   `current_voxel[3]`: Current traversal position [S1].
*   `target_voxel[3]`: End point voxel (LiDAR hit) [S1].

**Parameters**
*   `voxel_size`: $[0.2, 0.2, 0.2]$ meters [S1].
*   `point_cloud_range`: $[-51.2, -25.6, -2, 51.2, 25.6, 4.4]$ meters [S1].
*   `spatial_shape`: $[512, 256, 32]$ [S1].
*   `a_pre`/`a_post`: Frames to aggregate before/after for ground truth. Default: $70/70$ [S1].
*   `p_pre`/`p_post`: Input/prediction sequence lengths. Default: $10/10$ [S1].
*   `free_n_threshold`: $0$ [S1].
*   `occupied_n_threshold`: $1$ [S1].

**Steps**

1.  **Coordinate Transformation**: Normalize point origins using the inverse pose matrix and rotate points based on dataset-specific conventions. For Lyft, rotate $\pi$ around Z; for NuScenes, rotate $-\pi/2$ around Z [S1].
    $$
    \text{points}_{\text{norm}} = \text{pose\_origin\_inv} \times \text{points}
    $$
    $$
    \text{points}_{\text{rot}} = \text{points}_{\text{norm}} \times \text{rotation\_matrix}^T
    $$
    [S1]

2.  **Temporal Aggregation**: Aggregate input points from frames $[index - p_{pre}, index]$ and ground truth points from frames $[index, index + p_{post}]$ [S1]. For ground truth, further aggregate points from the window $[i - a_{pre}, i + a_{post}]$ for each target frame $i$ [S1].

3.  **Ray Initialization**: For each ray, calculate the direction vector from the sensor origin to the target point [S1]. Compute `step` ($+1$ or $-1$ per axis), `tDelta` (increment to move one voxel), and `tMax` (distance to the next voxel boundary) [S1].

4.  **Ray Traversal**: Traverse the ray path using a while loop with bounds checks [S1]. Mark the target voxel by incrementing `voxel_occ_count` using `atomicAdd` [S1]. Mark all traversed voxels (excluding the target) by incrementing `voxel_free_count` using `atomicAdd` [S1]. Stop rays at a distance of 1 voxel to prevent over-traversal, and ignore voxels too close to the sensor or target using a distance threshold of $1.0$m [S1].

5.  **Voxel State Assignment**: Assign states based on counts [S1]. If `voxel_occ_count` $> 1$, state = OCCUPIED ($1$) [S1]. Else if `voxel_free_count` $> 0$, state = FREE ($0$) [S1]. Else, state = NOT_OBSERVED ($-1$) [S1].

6.  **Label Assignment**: Calculate voxel indices for points within range [S1].
    $$
    \text{pcds\_voxel} = \lfloor (\text{points} - \text{range\_min}) / \text{voxel\_size} \rfloor
    $$
    Convert to scalar indices and assign point labels to `voxel_label_squeeze` [S1]. Default label for free voxels is $0$ [S1].

7.  **Invalid Mask Generation**: Generate a mask for voxels that are unobserved or have no label [S1].

8.  **Compression**: Compress binary occupancy data using bit packing [S1]. Reshape the boolean array $[H, W, D]$ to 1D, compress 8 bools into 1 `uint8`, and save as `.npy` [S1].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def ray_model_voxelization(points_origin, points, points_label, 
                           point_cloud_range, voxel_size, spatial_shape,
                           rotation_matrix=None, pose_origin_inv=None):
    """
    Convert point cloud to voxel grid using ray traversal.
    
    Args:
        points_origin: [N, 3] sensor origins
        points: [N, 3] LiDAR points
        points_label: [N] semantic labels
        point_cloud_range: [6] [xmin, ymin, zmin, xmax, ymax, zmax]
        voxel_size: [3] [vx, vy, vz]
        spatial_shape: [3] [H, W, D]
        rotation_matrix: [3, 3] optional rotation
        pose_origin_inv: [4, 4] optional inverse pose
        
    Returns:
        voxel_state: [H, W, D] int (-1, 0, 1)
        voxel_label: [H, W, D] int
        invalid_mask: [H, W, D] bool
    """
    # 1. Coordinate Transformation
    if pose_origin_inv is not None:
        points_h = np.hstack([points, np.ones((points.shape[0], 1))])
        points_norm = (pose_origin_inv @ points_h.T).T[:, :3]
    else:
        points_norm = points
        
    if rotation_matrix is not None:
        points_rot = points_norm @ rotation_matrix.T
    else:
        points_rot = points_norm
        
    points = points_rot
    
    # 2. Initialize Grids
    H, W, D = spatial_shape
    voxel_occ_count = np.zeros(spatial_shape, dtype=np.int32)
    voxel_free_count = np.zeros(spatial_shape, dtype=np.int32)
    voxel_label = np.zeros(spatial_shape, dtype=np.int32) # Default 0
    
    range_min = point_cloud_range[:3]
    
    # 3. Process Each Ray
    for i in range(points.shape[0]):
        origin = points_origin[i]
        target = points[i]
        
        # Skip if too close (1.0m threshold)
        dist = np.linalg.norm(target - origin)
        if dist < 1.0:
            continue
            
        ray_dir = (target - origin) / dist
        
        # Shift coordinates to voxel space origin
        origin_shifted = origin - range_min
        target_shifted = target - range_min
        
        # Calculate voxel indices for origin and target
        origin_voxel = np.floor(origin_shifted / voxel_size).astype(np.int32)
        target_voxel = np.floor(target_shifted / voxel_size).astype(np.int32)
        
        # Check bounds
        if not np.all(origin_voxel >= 0) or not np.all(origin_voxel < spatial_shape):
            continue
        if not np.all(target_voxel >= 0) or not np.all(target_voxel < spatial_shape):
            continue
            
        # Ray parameters in voxel space
        ray_dir_voxel = ray_dir / voxel_size
        step = np.where(ray_dir_voxel > 0, 1, -1).astype(np.int32)
        t_delta = 1.0 / np.abs(ray_dir_voxel)
        
        # tMax: distance to next boundary in t units
        t_max = np.zeros(3)
        for d in range(3):
            if ray_dir_voxel[d] > 0:
                t_max[d] = (origin_voxel[d] + 1) - (origin_shifted[d] / voxel_size[d])
            else:
                t_max[d] = origin_voxel[d] - (origin_shifted[d] / voxel_size[d])
                
        current_voxel = origin_voxel.copy()
        
        # Traverse ray
        # Stop 1 voxel before target to prevent over-traversal
        while not np.array_equal(current_voxel, target_voxel):
            # Mark free voxel (simulates atomicAdd) [S1]
            voxel_free_count[current_voxel[0], current_voxel[1], current_voxel[2]] += 1
            
            # Find next voxel
            t_next = np.argmin(t_max)
            current_voxel[t_next] += step[t_next]
            t_max[t_next] += t_delta[t_next]
            
            # Check bounds
            if not np.all(current_voxel >= 0) or not np.all(current_voxel < spatial_shape):
                break
                
        # Mark target voxel as occupied (simulates atomicAdd) [S1]
        voxel_occ_count[target_voxel[0], target_voxel[1], target_voxel[2]] += 1
        
    # 4. Assign States
    voxel_state = np.full(spatial_shape, -1, dtype=np.int32) # NOT_OBSERVED
    
    occupied_mask = voxel_occ_count > 1
    free_mask = (voxel_free_count > 0) & ~occupied_mask
    
    voxel_state[occupied_mask] = 1 # OCCUPIED
    voxel_state[free_mask] = 0 # FREE
    
    # 5. Assign Labels
    pcds_voxel = np.floor((points - range_min) / voxel_size).astype(np.int32)
    valid_points = np.all((pcds_voxel >= 0) & (pcds_voxel < spatial_shape), axis=1)
    
    if np.any(valid_points):
        valid_voxels = pcds_voxel[valid_points]
        valid_labels = points_label[valid_points]
        inds = valid_voxels[:, 0] * W * D + valid_voxels[:, 1] * D + valid_voxels[:, 2]
        voxel_label_flat = voxel_label.reshape(-1)
        voxel_label_flat[inds] = valid_labels
        voxel_label = voxel_label_flat.reshape(spatial_shape)
        
    # 6. Invalid Mask
    # Unobserved or occupied voxels with no label (default 0)
    invalid_mask = (voxel_state == -1) | ((voxel_state == 1) & (voxel_label == 0))
    
    # 7. Compression (for binary occupancy)
    # binary_occ = (voxel_state == 1)
    # compressed = np.packbits(binary_occ.reshape(-1))
    
    return voxel_state, voxel_label, invalid_mask
```

## Operating characteristics
*   **Performance**: Implemented using CUDA kernels with NVRTC for high performance [S1]. Atomic operations ensure thread safety [S1].
*   **Strengths**: Explicitly models free space, which is critical for occupancy prediction [S1]. Handles multi-frame temporal aggregation for dense ground truth [S1].
*   **Limitations**: Requires careful handling of coordinate transformations for different datasets [S1]. Numerical precision issues can occur if TF32 is not disabled in CUDA operations [S1].

## Background
The Ray Model Voxelization method is part of the Occ4cast project, specifically designed for processing LiDAR data for occupancy forecasting [S1]. The implementation is optimized for GPU execution using CUDA [S1].

## Sources
[S1] Ray Model and Voxelization | ai4ce/Occ4cast | DeepWiki, https://deepwiki.com/ai4ce/Occ4cast/3.2-ray-model-and-voxelization, fetched 2026-10-25
