# Point Cloud Cropping and Downsampling
Also called: Crop and Downsample Point Cloud

## Overview
Crops a point cloud using a specified volume and downsamples it using either voxel or uniform downsampling [S1, unofficial]. It differs from its siblings by providing a unified interface for spatial filtering and density reduction, which serves as a critical preprocessing step for downstream tasks such as ICP registration and voxelization [S1, unofficial]. Unlike [Farthest Point Sampling (FPS)](7182-farthest-point-sampling-fps.md) which operates in continuous Euclidean space to maximize geometric spread, this method relies on spatial binning or random selection within a defined bounding volume [S1, unofficial].

## How it works
**Inputs:**
- `points`: Array of point coordinates, shape $(N, 3)$.
- `crop_volume`: Bounding box or volume definition for cropping.
- `down_sample_method`: String, either `'voxel'` or `'uniform'`. Default: `'voxel'` [S1, unofficial].
- `voxel_size`: Float, size of the voxel grid cell for downsampling. Default: $0.01$ [S1, unofficial].
- `MAX_POINT_NUMBER`: Integer, maximum number of points before uniform downsampling is triggered. Value not specified in code snippet [S1, unofficial].
- `trans`: Optional transformation matrix applied before downsampling (used in registration variants) [S1, unofficial].

**Outputs:**
- `cropped_points`: Array of point coordinates after cropping and downsampling.

**Data Structures:**
- Points are stored as a dense array of 3D coordinates.
- Voxel grid is implicitly managed by hashing or binning coordinates into integer indices based on `voxel_size`.

**Steps:**
1. **Cropping:** Filter points that lie outside the `crop_volume`.
   - If `crop_volume` is an axis-aligned bounding box defined by min/max coordinates, retain points $p$ where $min_x \le p_x \le max_x$, $min_y \le p_y \le max_y$, and $min_z \le p_z \le max_z$.
2. **Downsampling:**
   - If `down_sample_method` is `'voxel'`:
     - Create a voxel grid with cell size `voxel_size`.
     - For each point, compute its voxel index: $idx = \lfloor p / voxel\_size \rfloor$.
     - Use a hash map or grid structure to store one representative point per occupied voxel (typically the first point encountered or the centroid of points in the voxel).
     - Return the set of representative points.
   - If `down_sample_method` is `'uniform'`:
     - Check if the number of points $N > MAX\_POINT\_NUMBER$.
     - If true, randomly select a subset of points such that the new count is $\le MAX\_POINT\_NUMBER$.
     - If false, return all points.
     - Note: `MAX_POINT_NUMBER` value is not specified in the source [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def crop_and_downsample(points, crop_volume, down_sample_method='voxel', voxel_size=0.01, MAX_POINT_NUMBER=None, trans=None):
    # Apply transformation if provided
    if trans is not None:
        points = apply_transformation(points, trans)
    
    # Step 1: Crop points within volume
    # Assuming crop_volume is a function or object that returns a boolean mask
    # For an AABB, this would be:
    # mask = (points >= crop_volume.min) & (points <= crop_volume.max)
    # cropped_points = points[mask]
    cropped_points = apply_crop(points, crop_volume)
    
    # Step 2: Downsample
    if down_sample_method == 'voxel':
        # Voxel Downsampling
        # Compute voxel indices
        voxel_indices = np.floor(cropped_points / voxel_size).astype(int)
        
        # Use a dictionary to store one point per voxel
        voxel_map = {}
        for i, point in enumerate(cropped_points):
            key = tuple(voxel_indices[i])
            if key not in voxel_map:
                voxel_map[key] = point
        
        # Return unique points
        return np.array(list(voxel_map.values()))
        
    elif down_sample_method == 'uniform':
        # Uniform Downsampling
        if MAX_POINT_NUMBER is None:
            # Default behavior if not specified
            return cropped_points
        
        if len(cropped_points) > MAX_POINT_NUMBER:
            # Randomly sample points
            indices = np.random.choice(len(cropped_points), MAX_POINT_NUMBER, replace=False)
            return cropped_points[indices]
        else:
            return cropped_points
    else:
        raise ValueError("Invalid down_sample_method")
```

**Parameters with Typical Values:**
- `voxel_size`: $0.01$ [S1, unofficial].
- `down_sample_method`: `'voxel'` [S1, unofficial].

**Edge Cases and Pitfalls:**
- `MAX_POINT_NUMBER` is not specified in the code snippet, so its default behavior is undefined [S1, unofficial].
- Precision issues may arise when computing voxel indices for points near boundaries; using `np.floor` is standard but may require careful handling of negative coordinates depending on the implementation [G].
- If `crop_volume` is empty or invalid, the output will be an empty point cloud.

## Subtopics

### Voxel Downsampling Registration
Performs ICP registration between source and ground truth target point clouds using voxel downsampling, with an initial transformation [S1, unofficial]. It differs from uniform registration by using voxel-based downsampling which preserves geometric structure better for dense clouds [S1, unofficial]. Choose this when high precision is required and the point cloud density is high.
1. If verbose: print `voxel_size` and threshold, set Open3D verbosity to Debug [S1, unofficial].
2. Downsample source: `crop_and_downsample(source, crop_volume, 'voxel', voxel_size, trans=init_trans)` [S1, unofficial].
3. Downsample target: `crop_and_downsample(gt_target, crop_volume, 'voxel', voxel_size)` [S1, unofficial].
4. Perform ICP: `registration_icp(s, t, threshold, np.identity(4), TransformationEstimationPointToPoint(True), ICPConvergenceCriteria(1e-6, max_itr))` [S1, unofficial].
5. Update transformation: `reg.transformation = matmul(reg.transformation, init_trans)` [S1, unofficial].
6. Return registration result [S1, unofficial].
Parameters: `voxel_size` is required [S1, unofficial]. ICP convergence: tolerance $1e-6$, max iterations `max_itr` [S1, unofficial].

### Uniform Registration
Performs ICP registration between source and ground truth target point clouds using uniform downsampling, with an initial transformation [S1, unofficial]. It differs from voxel registration by using random sampling, which is faster but may lose geometric detail [S1, unofficial]. Choose this when speed is prioritized over geometric fidelity.
1. If verbose: print threshold, set Open3D verbosity to Debug [S1, unofficial].
2. Downsample source: `crop_and_downsample(source, crop_volume, 'uniform', trans=init_trans)` [S1, unofficial].
3. Downsample target: `crop_and_downsample(gt_target, crop_volume, 'uniform')` [S1, unofficial].
4. Perform ICP: `registration_icp(s, t, threshold, np.identity(4), TransformationEstimationPointToPoint(True), ICPConvergenceCriteria(1e-6, max_itr))` [S1, unofficial].
5. Update transformation: `reg.transformation = matmul(reg.transformation, init_trans)` [S1, unofficial].
6. Return registration result [S1, unofficial].
Parameters: `max_size` is default $4 \times MAX\_POINT\_NUMBER$ but not used in current implementation [S1, unofficial]. ICP convergence: tolerance $1e-6$, max iterations `max_itr` [S1, unofficial].

### Point Cloud Filtering and Noise Removal
Removes clear floaters from unprojected point clouds for each frame using a 3D statistical filter from Open3D [S2]. It differs from cropping and downsampling by focusing on outlier removal rather than density reduction [S2]. Choose this when dealing with noisy unprojected point clouds.
Parameters: Number of neighbors: $20$. Standard deviation ratio: $2.0$ [S2].

## Operating characteristics
- **Time Complexity:** Voxel downsampling is $O(N)$ where $N$ is the number of points, assuming hash map operations are $O(1)$. Uniform downsampling is $O(N)$ for filtering and $O(K)$ for sampling where $K$ is the target size.
- **Memory Complexity:** $O(N)$ for storing points and voxel grid.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Voxel downsampling scales well with point cloud density as it reduces points to the number of occupied voxels.
- **Strengths:** Simple to implement, effective for reducing density while maintaining spatial distribution.
- **Limitations:** Voxel size selection is critical; too large a size loses detail, too small a size provides little reduction. Uniform downsampling may remove important geometric features.
- **When to use:** Preprocessing for ICP registration, voxelization, or rendering.
- **When not to use:** When geometric fidelity is critical and random sampling is unacceptable.

## Background
Originated from the Open3D library and used in 2D Gaussian Splatting implementations [S1, unofficial]. It is commonly used in 3D reconstruction and view synthesis pipelines to prepare point clouds for further processing [S1, unofficial][S2].

## Sources
[S1] hbb1/2d-gaussian-splatting, https://github.com/hbb1/2d-gaussian-splatting, fetched 2026-10-27 (unofficial)
[S2] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
