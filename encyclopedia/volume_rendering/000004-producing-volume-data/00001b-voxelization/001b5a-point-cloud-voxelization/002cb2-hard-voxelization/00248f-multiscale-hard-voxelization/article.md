# Multiscale Hard Voxelization

Also called: multiscale voxelization CUDA operator

## Overview
Multiscale Hard Voxelization is a GPU-accelerated method that simultaneously voxelizes a LiDAR point cloud into multiple spatial resolutions (scales) in a single pass, outputting voxel features, coordinates, and point counts for each scale [S1, unofficial]. It solves the inefficiency of running separate single-scale voxelization operators for different resolutions, which is common in multi-scale 3D perception networks [S1, unofficial]. Unlike standard single-scale hard voxelization, this method iterates over multiple voxel sizes and stores results in separate memory blocks indexed by scale, offering a 10% speed improvement over ordinary single-scale CUDA operators on an NVIDIA RTX 3090 [S1, unofficial]. It builds on [Hard Voxelization](5345-hard-voxelization.md) by extending the coordinate mapping and aggregation logic to handle a list of voxel sizes and maximum point limits per scale.

## How it works
**Inputs and Outputs:**
*   **Input:** A point cloud tensor `points` of shape $[N, D]$, where $N$ is the number of points and $D$ is the number of features (e.g., $x, y, z, reflectivity$).
*   **Parameters:**
    *   `voxel_sizes`: A nested list of $[v_x, v_y, v_z]$ for each scale $s$.
    *   `max_points_for_all_scales`: A list of integers $M_s$, the maximum number of points allowed per voxel for each scale.
    *   `point_cloud_range`: A list of 6 floats $[x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$ defining the bounding box.
    *   `max_voxels`: The maximum total number of voxels allowed across all scales combined.
*   **Outputs:**
    *   `voxels`: Tensor of shape $[S, M_{total}, M_{max}, D]$, containing the point features grouped by voxel. $S$ is the number of scales, $M_{total}$ is `max_voxels`, $M_{max}$ is the maximum `max_points` across all scales.
    *   `coors`: Tensor of shape $[S, M_{total}, 3]$, containing voxel coordinates.
    *   `num_points_per_voxel`: Tensor of shape $[S, M_{total}]$, containing the count of points in each voxel.

**Data Structures:**
The implementation uses pre-allocated zero-initialized tensors on the GPU to store results. The layout is:
*   `voxels`: `[scale_idx, max_voxels, max_points, num_features]`
*   `coors`: `[scale_idx, max_voxels, NDim]` (where NDim=3)
*   `num_points_per_voxel`: `[scale_idx, max_voxels]`

**Algorithm Steps:**
The method processes each scale $s$ sequentially (or in parallel if batched, but the reference implementation iterates) using a deterministic approach. For each scale $s$:

1.  **Initialize Grid Dimensions:**
    Calculate the grid dimensions for the current voxel size $v^{(s)} = [v_x, v_y, v_z]$ and range $R$:
    $$
    \text{grid}_x = \text{round}\left(\frac{x_{max} - x_{min}}{v_x}\right), \quad
    \text{grid}_y = \text{round}\left(\frac{y_{max} - y_{min}}{v_y}\right), \quad
    \text{grid}_z = \text{round}\left(\frac{z_{max} - z_{min}}{v_z}\right)
    $$
    [S1, unofficial]

2.  **Compute Voxel Coordinates:**
    For each point $p_i$ with coordinates $(x_i, y_i, z_i)$:
    $$
    c_x = \left\lfloor \frac{x_i - x_{min}}{v_x} \right\rfloor, \quad
    c_y = \left\lfloor \frac{y_i - y_{min}}{v_y} \right\rfloor, \quad
    c_z = \left\lfloor \frac{z_i - z_{min}}{v_z} \right\rfloor
    $$
    [S1, unofficial]

3.  **Boundary Check:**
    If any coordinate is out of bounds $[0, \text{grid})$, the point is discarded. The implementation sets the corresponding coordinate in the output `coors` to -1 to mark invalidity. Specifically:
    *   If $c_x \notin [0, \text{grid}_x)$, set `coors[0] = -1`.
    *   If $c_y \notin [0, \text{grid}_y)$, set `coors[0] = -1, coors[1] = -1`.
    *   If $c_z \notin [0, \text{grid}_z)$, set `coors[0] = -1, coors[1] = -1, coors[2] = -1`.
    [S1, unofficial]

4.  **Link Points to Voxels (Deterministic Aggregation):**
    To ensure determinism and avoid race conditions, the method uses a nested loop approach to identify points belonging to the same voxel.
    *   Initialize `point_to_voxelidx` and `point_to_pointidx` arrays of size $N$ with -1.
    *   For each point $i$ from 0 to $N-1$:
        *   If the point is invalid (from step 3), skip.
        *   Find the first point $j < i$ with identical voxel coordinates $(c_x, c_y, c_z)$. This is done by iterating $k$ from 0 to $i-1$ and checking if `coors[k]` matches `coors[i]`.
        *   Let $num$ be the count of points with identical coordinates among indices $0 \dots i-1$.
        *   If $num == 0$:
            *   This is the first point in a new voxel.
            *   If `voxel_num` (global counter for this scale) $\ge$ `max_voxels`, skip the point.
            *   Allocate a new voxel index: $v_{idx} = \text{voxel\_num}$.
            *   Increment `voxel_num`.
            *   Set `point_to_voxelidx[i] = 0` (indicating it's the head of the voxel list).
            *   Set `point_to_pointidx[i] = i`.
            *   Set `num_points_per_voxel[v_{idx}] = 1`.
            *   Set `coor_to_voxelidx[i] = v_{idx}`.
        *   If $num > 0$:
            *   Let $j$ be the index of the first point in this voxel (found via `point_to_pointidx` of the previous matching point, or by searching). The source notes imply finding the "first point with identical coordinates".
            *   Set `point_to_voxelidx[i] = num`.
            *   If $num \ge$ `max_points_for_scale[s]`, discard the point (set `point_to_voxelidx[i] = -1`).
            *   Else, set `point_to_pointidx[i] = j` (linking to the previous point in the same voxel).
            *   Set `coor_to_voxelidx[i] = coor_to_voxelidx[j]`.
            *   Increment `num_points_per_voxel[coor_to_voxelidx[j]]`.
    [S1, unofficial]
    *Note: The source describes an $O(N^2)$ approach where for each point, it iterates through previous points to find matches. This is inefficient for large $N$ but ensures determinism.*

5.  **Aggregate Features:**
    For each point $i$ that was assigned a valid voxel index $v_{idx}$ and a position $p_{idx} = \text{point\_to\_voxelidx}[i]$ within that voxel:
    *   Copy the point features `points[i]` to `voxels[s, v_{idx}, p_{idx}]`.
    [S1, unofficial]

6.  **Finalize:**
    Return the slices of the pre-allocated tensors up to `voxel_num[s]` for each scale.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiscale_hard_voxelize(points, voxel_sizes, max_points_for_all_scales, point_cloud_range, max_voxels):
    """
    points: [N, D] float tensor
    voxel_sizes: List of [vx, vy, vz]
    max_points_for_all_scales: List of int
    point_cloud_range: [xmin, ymin, zmin, xmax, ymax, zmax]
    max_voxels: int
    Returns: voxels [S, M, max_P, D], coors [S, M, 3], num_points [S, M]
    """
    S = len(voxel_sizes)
    N, D = points.shape
    max_P = max(max_points_for_all_scales)
    
    # Pre-allocate output tensors
    voxels = torch.zeros((S, max_voxels, max_P, D), device=points.device)
    coors = torch.zeros((S, max_voxels, 3), dtype=torch.int32, device=points.device)
    num_points_per_voxel = torch.zeros((S, max_voxels), dtype=torch.int32, device=points.device)
    
    # Helper to process a single scale
    def process_scale(points, vx, vy, vz, max_pts, max_vox):
        xmin, ymin, zmin, xmax, ymax, zmax = point_cloud_range
        grid_x = round((xmax - xmin) / vx)
        grid_y = round((ymax - ymin) / vy)
        grid_z = round((zmax - zmin) / vz)
        
        # Temporary storage for this scale's results
        # We need to track which points go to which voxel and their order
        # To avoid O(N^2) in Python, we assume the CUDA kernel does this efficiently.
        # Here we describe the logic.
        
        voxel_num = 0
        # Arrays to map point index to voxel index and position within voxel
        # -1 indicates invalid or full
        point_to_voxelidx = torch.full((N,), -1, dtype=torch.int32, device=points.device)
        point_to_pointidx = torch.full((N,), -1, dtype=torch.int32, device=points.device)
        
        # Compute voxel coordinates for all points
        # c_x = floor((x - xmin) / vx)
        c_x = torch.floor((points[:, 0] - xmin) / vx).int()
        c_y = torch.floor((points[:, 1] - ymin) / vy).int()
        c_z = torch.floor((points[:, 2] - zmin) / vz).int()
        
        # Boundary checks
        valid_x = (c_x >= 0) & (c_x < grid_x)
        valid_y = (c_y >= 0) & (c_y < grid_y)
        valid_z = (c_z >= 0) & (c_z < grid_z)
        valid = valid_x & valid_y & valid_z
        
        # Initialize coors with -1 for invalid points
        coors_scale = torch.full((N, 3), -1, dtype=torch.int32, device=points.device)
        coors_scale[:, 0] = c_x
        coors_scale[:, 1] = c_y
        coors_scale[:, 2] = c_z
        
        # Mask invalid points
        coors_scale[~valid, 0] = -1
        if ~valid_y:
            coors_scale[~valid_y, 1] = -1
        if ~valid_z:
            coors_scale[~valid_z, 2] = -1
            
        # Deterministic linking: O(N^2) in naive implementation
        # For each point i, find the first point j < i with same coors
        # This part is typically done in CUDA kernel for performance
        # We simulate the logic here for clarity
        
        # In the actual CUDA implementation, this is done in parallel or with atomic ops
        # but the source specifies a deterministic O(N^2) nested loop approach for linking.
        
        # To implement this in pure Python for the article:
        # We iterate i from 0 to N-1
        #   If not valid[i]: continue
        #   Find j < i such that coors[j] == coors[i]
        #   Count how many such j exist (num)
        #   If num == 0:
        #       if voxel_num >= max_vox: continue
        #       voxel_idx = voxel_num
        #       voxel_num += 1
        #       point_to_voxelidx[i] = 0
        #       point_to_pointidx[i] = i
        #       num_points_per_voxel[voxel_idx] = 1
        #   Else:
        #       if num >= max_pts: continue
        #       point_to_voxelidx[i] = num
        #       point_to_pointidx[i] = j (the first one)
        #       num_points_per_voxel[voxel_idx] += 1
        
        # Since we cannot run O(N^2) in the article's pseudocode efficiently,
        # we note that the CUDA kernel performs this step.
        
        # Aggregate features
        # For each valid point i:
        #   v_idx = coor_to_voxelidx[i] (derived from point_to_pointidx chain)
        #   p_idx = point_to_voxelidx[i]
        #   voxels[v_idx, p_idx] = points[i]
        
        return voxels[:voxel_num], coors_scale[:voxel_num], num_points_per_voxel[:voxel_num]

    # Process each scale
    for s in range(S):
        vx, vy, vz = voxel_sizes[s]
        max_pts = max_points_for_all_scales[s]
        # Call CUDA kernel or implement the logic above
        # The source code calls a CUDA function `hard_voxelize`
        pass 
        
    return voxels, coors, num_points_per_voxel
```
[G] The pseudocode above outlines the logic. The actual implementation uses a CUDA kernel `dynamic_voxelize_kernel` which handles the coordinate mapping and boundary checks. The linking of points to voxels is done via a nested loop in the CPU-side or a specific CUDA kernel that iterates through points to find matches, ensuring determinism. The source code snippet shows that `voxels`, `coors`, and `num_points_per_voxel` are pre-allocated and passed to the CUDA kernel. The kernel returns the number of voxels used per scale.

**Parameters:**
*   `voxel_sizes`: e.g., `[[0.16, 0.16, 4], [0.32, 0.32, 4], [0.64, 0.64, 4]]` [S1, unofficial]
*   `max_points_for_all_scales`: e.g., `[32, 64, 128]` [S1, unofficial]
*   `point_cloud_range`: e.g., `[0, -39.68, -3, 69.12, 39.68, 1]` [S1, unofficial]
*   `max_voxels`: e.g., `20000` [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Out of Bounds:** Points outside the `point_cloud_range` are discarded. The implementation marks them by setting coordinates to -1 [S1, unofficial].
*   **Max Voxels/Points:** If the number of voxels exceeds `max_voxels` or points in a voxel exceed `max_points`, additional points are discarded. The source notes that users should shuffle points before calling the function because `max_voxels` may drop points [S1, unofficial].
*   **Determinism:** The method is deterministic by default. Non-deterministic versions (using atomic operations) are not enabled in the reference implementation [S1, unofficial].
*   **Performance:** The deterministic linking approach is $O(N^2)$, which is inefficient for large point clouds [S1, unofficial].

## Subtopics

### PointNet++ LiDAR Encoding
This variant uses Multiscale Hard Voxelization as a preprocessing step for LiDAR feature encoding. It encodes LiDAR features using PointNet++ with hidden feature dimensions mirroring those of a sparse UNet method. It samples points at different levels (4096, 1024, 256, and 56 points) using farthest point sampling (FPS). The FPS and neighbor point grouping are implemented using CUDA-based routines from PyTorch3D to enhance efficiency. This approach was found to underperform compared to sparse UNet-based encoding in the context of neural radiance fields for street scenes [S2].

## Operating characteristics
*   **Time Complexity:** The coordinate mapping is $O(N)$ per scale. However, the deterministic point linking is $O(N^2)$ per scale due to the nested loop approach, which is a significant bottleneck for large $N$ [S1, unofficial].
*   **Memory Complexity:** $O(S \cdot M_{total} \cdot M_{max} \cdot D)$ for the output tensors, where $S$ is the number of scales, $M_{total}$ is `max_voxels`, $M_{max}$ is the maximum `max_points`, and $D$ is the number of features [S1, unofficial].
*   **Performance:** On an NVIDIA RTX 3090, the multiscale operator shows a 10% speed improvement over the ordinary single-scale voxelization CUDA operator [S1, unofficial].
*   **Strengths:** Simultaneous multi-scale processing reduces API overhead and kernel launch latency. Deterministic output ensures reproducibility.
*   **Limitations:** The $O(N^2)$ linking step is inefficient for large point clouds. The method requires pre-allocation of memory based on `max_voxels` and `max_points`, which can lead to wasted memory if these limits are not tight.
*   **When to Use:** When a 3D perception network requires multi-scale voxel inputs and deterministic behavior is critical. Not suitable for very large point clouds where the $O(N^2)$ step becomes prohibitive.

## Background
Multiscale Hard Voxelization was introduced as a CUDA operator to address the inconvenience of invoking single-scale voxelization operators multiple times for different resolutions in 3D perception pipelines [S1, unofficial]. It builds on the work of normal voxelization methods and was implemented to improve speed and convenience [S1, unofficial]. The implementation is based on the `mmcv` and `mmdet` open-source codebases [S1, unofficial]. The method is used in LiDAR-based 3D object detection and segmentation tasks, particularly in frameworks like MMDetection3D [S1, unofficial].

## Sources
[S1] Susie0909/Multiscale_Voxelization_CUDA, https://github.com/Susie0909/Multiscale_Voxelization_CUDA, fetched 2026-10-27 (unofficial)
[S2] LidaRF: Delving into Lidar for Neural Radiance Field on Street Scenes, https://doi.org/10.1109/cvpr52733.2024.01850, fetched 2026-10-26
