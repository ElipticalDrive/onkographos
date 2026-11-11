# Multi-Plane Point Cloud Voxelization and Weight Aggregation

Also called: MPP Aggregation, MPP Voxelization, Pre-processing for Neural Point Cloud Rendering via Multi-Plane Projection

## Overview
This method is a pre-processing pipeline that projects a 3D point cloud onto multiple parallel depth planes (voxels), groups points by their projected 2D grid location and depth bin, and computes aggregation weights for rendering [S1, unofficial]. It solves the problem of efficiently organizing sparse point cloud data for multi-plane projection rendering by reducing outlier influence and grouping points spatially [S1, unofficial]. It differs from [DCX Points-to-Voxels](3783-dcx-points-to-voxels.md) by explicitly using a multi-plane depth discretization strategy with learned/weighted aggregation, and from [VoxelGrid::CreateFromPointCloudWithinBounds](8505-voxelgrid-createfrompointcloudwithinbounds.md) by handling depth-based plane assignment and weight normalization rather than simple spatial occupancy.

## How it works
**Inputs:**
- Point cloud file (PLY format) containing positions and colors.
- Camera intrinsics and extrinsics (text files: `intrinsic_color.txt`, `intrinsic_depth.txt`, `extrinsic_color.txt`, `extrinsic_depth.txt`, `index.txt`).
- Parameters: `num_planes` (default 32), `a` (default 1), `b` (default 1), target image resolution (e.g., 640x480), original resolution (e.g., 1296x968 for color, 640x480 for depth), depth bounds (`valid_depth_near`, `valid_depth_far`).

**Outputs:**
- NPZ file containing:
  - `u`, `v`: float32, pixel coordinates.
  - `d`: float32, depth.
  - `select_index`: int64, original point indices.
  - `group_belongs`: int64, group ID.
  - `index_in_each_group`: uint16, index within group.
  - `distance`: float32, parallel distance.
  - `each_split_max_num`: uint16, max points per plane.
  - `delta`: float32, plane depth differences.

**Data Structures:**
- Points are stored as arrays of positions `[x, y, z, 1.0]` and colors `[b, g, r]`.
- Camera matrices are 4x4 float32 numpy arrays.
- Group keys are computed as `u + v * w + d_position * w * h`.

**Steps:**

1. **Load Point Cloud:**
   - Parse PLY file until `end_header`.
   - Read lines, splitting by space.
   - Append position as `[x, y, z, 1.0]` and color as `[b, g, r]` (swapping RGB to BGR).
   - Return transposed numpy arrays for positions and colors.

2. **Read Camera Parameters:**
   - Read intrinsic and extrinsic matrices from text files.
   - Each file contains 4 lines of space-separated floats, converted to 4x4 numpy float32 arrays.
   - Adjust extrinsic matrix: `extrinsic_matrix[:, [1, 2]] = extrinsic_matrix[:, [1, 2]] * (-1.0)` for camera coordinate system transformation.

3. **Adjust Intrinsics for Resizing:**
   - If image resizing is enabled, calculate scale factors:
     - `scale1_c = (1296 - 1) / (target_w - 1)`
     - `scale2_c = (968 - 1) / (target_h - 1)` for color.
     - `scale1_d`, `scale2_d` for depth.
   - Divide the first two rows of the intrinsic matrices by these scales to map from original resolution to target resolution.

4. **Project Point Cloud to Image Space:**
   - Compute `transform_matrix = intrinsic_matrix.dot(np.linalg.inv(extrinsic_matrix))`.
   - Compute `position_image = transform_matrix.dot(point_clouds)`.
   - Extract `depth_all = position_image[2, :]`.
   - Compute `u_all = position_image[0, :] / (depth_all + 1e-10)`.
   - Compute `v_all = position_image[1, :] / (depth_all + 1e-10)`.

5. **Filter Valid Points:**
   - Identify `valid_u` where `0 <= u_all <= w-1`.
   - Identify `valid_v` where `0 <= v_all <= h-1`.
   - Identify `valid_d` where `valid_depth_near < depth_all < valid_depth_far`.
   - Intersect these indices to get `valid_position`.
   - Sort selected depths in descending order using `np.argsort(-selected_depth)`.
   - Remove the 100 furthest and 50 nearest points to reduce outlier influence: `index = index[100:-50]`.

6. **Calculate Parallel Distance to Grid Center:**
   - Round `u` and `v` to nearest integer: `u_sorted = np.uint32(np.rint(center_u_sorted))`, `v_sorted = np.uint32(np.rint(center_v_sorted))`.
   - Compute `distance_sorted = sqrt((u_sorted - center_u_sorted)^2 + (v_sorted - center_v_sorted)^2)`.

7. **Define Depth Bins for Planes:**
   - Get `valid_d_min` (nearest depth) and `valid_d_max` (farthest depth) from sorted depths.
   - Create `num_planes+1` linearly spaced boundaries: `depth_tmp = np.linspace(valid_d_max, valid_d_min, num_planes+1)`.
   - Randomly sample one boundary per plane bin: `tmp[bin] = random.uniform(depth_tmp[bin], depth_tmp[bin+1])`.
   - Compute `delta[bin] = tmp[bin] - tmp[bin+1]` for `bin` in `range(num_planes-1)`, then flip `delta`.

8. **Assign Points to Planes:**
   - Initialize `up_boundary = tmp`.
   - Iterate through sorted points.
   - For each point with depth `tmp_d`, find the plane index `d_position[i] = num_planes - cnt - 1` such that `tmp_d >= up_boundary[cnt]`.
   - Increment `cnt` and check subsequent boundaries until the condition is met. Break when found.

9. **Group Points by Voxel:**
   - Compute group key: `groups_original = u_sorted + v_sorted * w + d_position * w * h`.
   - Sort points by this key.
   - Assign group IDs: iterate through sorted groups, when group key changes, assign current group ID to all points in the previous group, store start index in `groups_index`, and reset `group_begin`.
   - Use `array[0:(ii+1 - group_begin)]` to assign sequential indices within each group.

10. **Compute Aggregation Weights:**
    - Calculate `weight_1 = (1 - distance)^a`.
    - Calculate `weight_2 = 1 / (1 + distance_1)^b`, where `distance_1` is the vertical distance to the minimum depth in the group.
    - Compute `weight_renew = weight_1 * weight_2`.
    - Normalize weights per group by dividing by the sum of weights in each group.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import random

def load_ply(ply_path):
    position = []
    color = []
    with open(ply_path, 'r') as f:
        begin = False
        for line in f:
            line = line.strip('\n').split(' ')
            if not line:
                continue
            if line[0] == 'end_header':
                begin = True
                continue
            if begin:
                position.append([float(line[0]), float(line[1]), float(line[2]), 1.0])
                color.append([float(line[5]), float(line[4]), float(line[3])]) # RGB to BGR
    return np.array(position).T, np.array(color).T

def read_camera_params(intrinsic_path, extrinsic_path):
    def read_matrix(path):
        lines = []
        with open(path, 'r') as f:
            for line in f:
                lines.extend(line.strip().split())
        mat = np.array(lines, dtype=np.float32).reshape(4, 4)
        return mat
    intrinsic = read_matrix(intrinsic_path)
    extrinsic = read_matrix(extrinsic_path)
    extrinsic[:, [1, 2]] = extrinsic[:, [1, 2]] * (-1.0)
    return intrinsic, extrinsic

def adjust_intrinsics(intrinsic, orig_w, orig_h, target_w, target_h):
    scale1 = (orig_w - 1) / (target_w - 1)
    scale2 = (orig_h - 1) / (target_h - 1)
    intrinsic[0, :] /= scale1
    intrinsic[1, :] /= scale2
    return intrinsic

def project_points(positions, intrinsic, extrinsic):
    transform_matrix = intrinsic.dot(np.linalg.inv(extrinsic))
    position_image = transform_matrix.dot(positions)
    depth_all = position_image[2, :]
    u_all = position_image[0, :] / (depth_all + 1e-10)
    v_all = position_image[1, :] / (depth_all + 1e-10)
    return u_all, v_all, depth_all

def filter_points(u_all, v_all, depth_all, w, h, valid_depth_near, valid_depth_far):
    valid_u = (u_all >= 0) & (u_all < w)
    valid_v = (v_all >= 0) & (v_all < h)
    valid_d = (depth_all > valid_depth_near) & (depth_all < valid_depth_far)
    valid_mask = valid_u & valid_v & valid_d
    valid_indices = np.where(valid_mask)[0]
    selected_depths = depth_all[valid_indices]
    sorted_indices = np.argsort(-selected_depths)
    # Remove 100 furthest and 50 nearest
    if len(sorted_indices) > 150:
        sorted_indices = sorted_indices[100:-50]
    return valid_indices[sorted_indices]

def calculate_distance(u_sorted, v_sorted, center_u, center_v):
    return np.sqrt((u_sorted - center_u)**2 + (v_sorted - center_v)**2)

def define_depth_bins(valid_d_min, valid_d_max, num_planes):
    depth_tmp = np.linspace(valid_d_max, valid_d_min, num_planes + 1)
    tmp = np.zeros(num_planes)
    for bin_idx in range(num_planes):
        tmp[bin_idx] = random.uniform(depth_tmp[bin_idx], depth_tmp[bin_idx + 1])
    delta = np.zeros(num_planes - 1)
    for bin_idx in range(num_planes - 1):
        delta[bin_idx] = tmp[bin_idx] - tmp[bin_idx + 1]
    delta = np.flip(delta)
    return tmp, delta

def assign_planes(depth_all, tmp, num_planes):
    up_boundary = tmp
    d_position = np.zeros_like(depth_all, dtype=np.int32)
    for i, d in enumerate(depth_all):
        cnt = 0
        while cnt < num_planes:
            if d >= up_boundary[cnt]:
                d_position[i] = num_planes - cnt - 1
                break
            cnt += 1
    return d_position

def group_points(u_sorted, v_sorted, d_position, w, h):
    groups_original = u_sorted + v_sorted * w + d_position * w * h
    sort_indices = np.argsort(groups_original)
    sorted_groups = groups_original[sort_indices]
    group_ids = np.zeros(len(sorted_groups), dtype=np.int64)
    index_in_group = np.zeros(len(sorted_groups), dtype=np.uint16)
    current_group_id = 0
    group_begin = 0
    for i in range(1, len(sorted_groups)):
        if sorted_groups[i] != sorted_groups[i-1]:
            group_ids[group_begin:i] = current_group_id
            index_in_group[group_begin:i] = np.arange(i - group_begin, dtype=np.uint16)
            current_group_id += 1
            group_begin = i
    group_ids[group_begin:] = current_group_id
    index_in_group[group_begin:] = np.arange(len(sorted_groups) - group_begin, dtype=np.uint16)
    return sort_indices, group_ids, index_in_group

def compute_weights(distance_sorted, depth_all, group_ids, index_in_group, a, b):
    weights = np.ones_like(distance_sorted)
    for gid in np.unique(group_ids):
        mask = group_ids == gid
        group_depths = depth_all[mask]
        min_depth = np.min(group_depths)
        distance_1 = group_depths - min_depth
        weight_1 = (1 - distance_sorted[mask]) ** a
        weight_2 = 1 / (1 + distance_1) ** b
        weight_renew = weight_1 * weight_2
        weights[mask] = weight_renew
    # Normalize per group
    for gid in np.unique(group_ids):
        mask = group_ids == gid
        group_weights = weights[mask]
        weights[mask] = group_weights / np.sum(group_weights)
    return weights

def mpp_voxelization(ply_path, intrinsic_color_path, extrinsic_color_path, w, h, valid_depth_near, valid_depth_far, num_planes=32, a=1, b=1):
    positions, colors = load_ply(ply_path)
    intrinsic, extrinsic = read_camera_params(intrinsic_color_path, extrinsic_color_path)
    intrinsic = adjust_intrinsics(intrinsic, 1296, 968, w, h)
    u_all, v_all, depth_all = project_points(positions, intrinsic, extrinsic)
    valid_indices = filter_points(u_all, v_all, depth_all, w, h, valid_depth_near, valid_depth_far)
    u_valid = u_all[valid_indices]
    v_valid = v_all[valid_indices]
    depth_valid = depth_all[valid_indices]
    center_u = w / 2
    center_v = h / 2
    u_sorted = np.uint32(np.rint(u_valid))
    v_sorted = np.uint32(np.rint(v_valid))
    distance_sorted = calculate_distance(u_sorted, v_sorted, center_u, center_v)
    valid_d_min = np.min(depth_valid)
    valid_d_max = np.max(depth_valid)
    tmp, delta = define_depth_bins(valid_d_min, valid_d_max, num_planes)
    d_position = assign_planes(depth_valid, tmp, num_planes)
    sort_indices, group_ids, index_in_group = group_points(u_sorted, v_sorted, d_position, w, h)
    weights = compute_weights(distance_sorted, depth_valid, group_ids, index_in_group, a, b)
    # Output structure
    output = {
        'u': u_valid[sort_indices],
        'v': v_valid[sort_indices],
        'd': depth_valid[sort_indices],
        'select_index': valid_indices[sort_indices],
        'group_belongs': group_ids,
        'index_in_each_group': index_in_group,
        'distance': distance_sorted[sort_indices],
        'each_split_max_num': np.bincount(d_position[sort_indices]).max(),
        'delta': delta
    }
    return output
```

**Parameters:**
- `num_planes`: Number of planes used for voxelization. Default value is 32 [S1, unofficial].
- `a`: Hyperparameter for parallel distance weighting. Default value is 1 [S1, unofficial].
- `b`: Hyperparameter for vertical distance weighting. Default value is 1 [S1, unofficial].

**Edge Cases and Pitfalls:**
- Invalid camera poses with `-inf` values cause crashes. Check if `extrinsic_matrix[0, 0] < -1e6` and skip such frames [S1, unofficial].
- Division by zero in depth normalization. Add epsilon `1e-10` to `depth_all` before dividing `u` and `v` [S1, unofficial].

## Subtopics

### [Multi-Plane Image (MPI) Rendering from Point Clouds](12104-multi-plane-image-mpi-rendering-from-point-clouds.md)
A preprocessing technique to render a 3D point cloud into a set of 2D image planes (MPI) by projecting points onto a target image grid and handling occlusion between planes. This method provides the voxelization and weight aggregation step that feeds into the MPI rendering process.

## Operating characteristics
- **Time Complexity:** Dominated by sorting points by depth and group keys, which is $O(N \log N)$ where $N$ is the number of valid points. Plane assignment and weight computation are linear $O(N)$.
- **Memory Complexity:** $O(N)$ to store point attributes, group IDs, and weights. The output NPZ file size depends on $N$ and the number of planes.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of points. The number of planes is a fixed parameter.
- **Strengths:** Reduces outlier influence by trimming nearest and furthest points. Groups points spatially and by depth, facilitating efficient rendering.
- **Limitations:** Relies on accurate camera parameters. Outlier removal is heuristic (fixed number of points).
- **When to use:** When preparing point clouds for multi-plane projection rendering where spatial grouping and depth-based weighting are beneficial.
- **When not to use:** When real-time voxelization is required without pre-processing, or when the point cloud is too sparse for effective plane assignment.

## Background
This method originates from the PCMP-NeRF project, an unofficial implementation for neural point cloud rendering [S1, unofficial]. It is used as a pre-processing step to organize point cloud data for efficient rendering via multi-plane images.

## Sources
[S1] PCMP-NeRF, https://github.com/Mayxmu/PCMP-NeRF, fetched 2026-10-30 (unofficial)
[S2] PCMP_NeRF, https://github.com/Mayxmu/PCMP_NeRF, fetched 2026-10-30 (unofficial)
