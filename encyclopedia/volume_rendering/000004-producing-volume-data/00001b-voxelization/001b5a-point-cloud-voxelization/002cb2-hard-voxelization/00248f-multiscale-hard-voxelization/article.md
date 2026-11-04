# Multiscale Hard Voxelization
## Overview
Multiscale Hard Voxelization is a CUDA-based operator that voxelizes a LiDAR point cloud into multiple spatial scales simultaneously, outputting voxel features, coordinates, and point counts for each scale [S1, unofficial]. It differs from standard single-scale voxelization by iterating over multiple scales and storing results in separate memory blocks indexed by `scale_idx`, providing convenience for multi-scale feature extraction networks [S1, unofficial].
## How it works
**Inputs and Outputs**
- **Input**: A point cloud tensor of shape $[N, D]$, where $N$ is the number of points and $D$ is the number of features (typically 4: $x, y, z, \text{reflectivity}$) [S1, unofficial].
- **Parameters**:
  - `voxel_sizes`: A nested list defining voxel dimensions $[x, y, z]$ for each scale (e.g., `[[0.16, 0.16, 4], [0.32, 0.32, 4]]`) [S1, unofficial].
  - `max_points_for_all_scales`: A list defining the maximum number of points allowed per voxel for each scale (e.g., `[32, 64]`) [S1, unofficial].
  - `point_cloud_range`: A list of 6 floats $[x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$ defining the bounding box of the voxel grid in meters [S1, unofficial].
  - `max_voxels`: The maximum total number of voxels allowed across all scales [S1, unofficial].
- **Outputs**:
  - `voxels`: Tensor of shape $[S, M, P_{max}, D]$, containing the point features within each voxel, where $S$ is the number of scales, $M$ is `max_voxels`, and $P_{max}$ is the maximum points per voxel across all scales [S1, unofficial].
  - `coors`: Tensor of shape $[S, M, 3]$, containing voxel coordinates in $[z, y, x]$ order [S1, unofficial].
  - `num_points_per_voxel`: Tensor of shape $[S, M]$, containing the count of points in each voxel [S1, unofficial].

**Data Structures**
The implementation uses pre-allocated tensors on the GPU:
1. `voxels`: $[S, \text{max\_voxels}, \max(\text{max\_points}), D]$ initialized to zeros [S1, unofficial].
2. `coors`: $[S, \text{max\_voxels}, 3]$ initialized to zeros [S1, unofficial].
3. `num_points_per_voxel`: $[S, \text{max\_voxels}]$ initialized to zeros [S1, unofficial].
4. `voxel_num`: An array of size $S$ tracking the number of allocated voxels per scale [S1, unofficial].
5. `point_to_pointidx`: An array of size $N$ mapping each point to the index of the previous point in the same voxel [S1, unofficial].
6. `point_to_voxelidx`: An array of size $N$ mapping each point to its position (index) within its voxel (0-based) [S1, unofficial].
7. `coor_to_voxelidx`: An array of size $N$ mapping each point to the global voxel index [S1, unofficial].

**Algorithm Steps**
The operator processes each scale $s$ from $0$ to $S-1$. For each scale, it performs the following steps for every point $i$ in the point cloud:

1. **Grid Dimension Calculation**:
   Calculate the grid dimensions for the current scale $s$ using the voxel size $v_s = [v_x, v_y, v_z]$ and range $R = [x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$:
   $$
   \text{grid}_x = \left\lfloor \frac{x_{max} - x_{min}}{v_x} \right\rfloor, \quad
   \text{grid}_y = \left\lfloor \frac{y_{max} - y_{min}}{v_y} \right\rfloor, \quad
   \text{grid}_z = \left\lfloor \frac{z_{max} - z_{min}}{v_z} \right\rfloor
   $$
   [S1, unofficial]

2. **Voxel Coordinate Computation**:
   For each point $i$ with coordinates $p_i = [x_i, y_i, z_i]$, compute the voxel coordinates $c_i = [c_x, c_y, c_z]$:
   $$
   c_x = \left\lfloor \frac{x_i - x_{min}}{v_x} \right\rfloor, \quad
   c_y = \left\lfloor \frac{y_i - y_{min}}{v_y} \right\rfloor, \quad
   c_z = \left\lfloor \frac{z_i - z_{min}}{v_z} \right\rfloor
   $$
   [S1, unofficial]

3. **Boundary Check**:
   If any coordinate is out of bounds, the point is discarded for this scale:
   - If $c_x < 0$ or $c_x \ge \text{grid}_x$, set $c_x = -1$ and return.
   - If $c_y < 0$ or $c_y \ge \text{grid}_y$, set $c_y = -1$ and return.
   - If $c_z < 0$ or $c_z \ge \text{grid}_z$, set $c_z = -1$ and return.
   [S1, unofficial]

4. **Point Linking (Deterministic)**:
   To handle points falling into the same voxel, the implementation uses a deterministic $O(N^2)$ approach by iterating backwards from the current point to find the previous point with identical coordinates [S1, unofficial].
   - Initialize `num = 0`.
   - Iterate $j$ from $i-1$ down to $0$:
     - If $c_j == c_i$ (coordinates match), increment `num`.
     - If `num == 1`, set `point_to_pointidx[i] = j`.
     - If `num >= max_points[s]`, stop and discard point $i$ (return).
   [S1, unofficial]

5. **Voxel Index Assignment**:
   - Set `point_to_voxelidx[i] = num` (if not discarded).
   - If `num == 0`:
     - This is the first point in a new voxel.
     - If `voxel_num[s] >= max_voxels`, skip point $i$.
     - Allocate a new voxel index: `voxelidx = voxel_num[s]`.
     - Increment `voxel_num[s]`.
     - Set `coor_to_voxelidx[i] = voxelidx`.
     - Set `num_points_per_voxel[voxelidx] = 1`.
     - Set `point_to_pointidx[i] = i`.
   - If `num > 0`:
     - This point belongs to an existing voxel.
     - Find the first point in the same voxel: `point_idx = point_to_pointidx[i]`.
     - Set `coor_to_voxelidx[i] = coor_to_voxelidx[point_idx]`.
     - Increment `num_points_per_voxel[coor_to_voxelidx[i]]`.
   [S1, unofficial]

6. **Data Storage**:
   - Store the voxel coordinates in `coors[s, voxelidx]` as $[c_z, c_y, c_x]$ [S1, unofficial].
   - Store the point features in `voxels[s, voxelidx, num]` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def multiscale_hard_voxelize(points, voxel_sizes, max_points_for_all_scales, point_cloud_range, max_voxels):
    """
    points: [N, D] float tensor
    voxel_sizes: List[List[float]] of shape [S, 3]
    max_points_for_all_scales: List[int] of shape [S]
    point_cloud_range: List[float] of shape [6] [x_min, y_min, z_min, x_max, y_max, z_max]
    max_voxels: int
    Returns: voxels [S, M, P_max, D], coors [S, M, 3], num_points_per_voxel [S, M]
    """
    S = len(voxel_sizes)
    N = points.shape[0]
    D = points.shape[1]
    P_max = max(max_points_for_all_scales)
    
    # Initialize output tensors
    voxels = torch.zeros((S, max_voxels, P_max, D), dtype=points.dtype, device=points.device)
    coors = torch.zeros((S, max_voxels, 3), dtype=torch.int32, device=points.device)
    num_points_per_voxel = torch.zeros((S, max_voxels), dtype=torch.int32, device=points.device)
    
    # Helper arrays for linking points (allocated per scale or reused)
    # Note: In the actual CUDA implementation, these are passed to the kernel.
    # Here we simulate the logic for clarity.
    
    voxel_num = torch.zeros(S, dtype=torch.int32, device=points.device)
    
    for s in range(S):
        vx, vy, vz = voxel_sizes[s]
        max_p = max_points_for_all_scales[s]
        x_min, y_min, z_min, x_max, y_max, z_max = point_cloud_range
        
        grid_x = int((x_max - x_min) / vx)
        grid_y = int((y_max - y_min) / vy)
        grid_z = int((z_max - z_min) / vz)
        
        # Allocate temporary arrays for this scale
        point_to_pointidx = torch.full((N,), -1, dtype=torch.int32, device=points.device)
        point_to_voxelidx = torch.full((N,), -1, dtype=torch.int32, device=points.device)
        coor_to_voxelidx = torch.full((N,), -1, dtype=torch.int32, device=points.device)
        
        for i in range(N):
            x, y, z = points[i, 0], points[i, 1], points[i, 2]
            
            # Compute voxel coordinates
            c_x = int((x - x_min) / vx)
            c_y = int((y - y_min) / vy)
            c_z = int((z - z_min) / vz)
            
            # Boundary check
            if c_x < 0 or c_x >= grid_x:
                continue
            if c_y < 0 or c_y >= grid_y:
                continue
            if c_z < 0 or c_z >= grid_z:
                continue
            
            # Find previous point with identical coordinates
            num = 0
            point_to_pointidx[i] = i # Default to self if no previous match
            for j in range(i - 1, -1, -1):
                # Check if point j has same coordinates
                # Optimization: only check if j is not discarded and has valid coors
                # For simplicity, we assume we can access points[j]
                if points[j, 0] == x and points[j, 1] == y and points[j, 2] == z:
                    # Check if j is in the same voxel (it should be if coords match)
                    # But we need to ensure j was processed for this scale
                    # In the CUDA kernel, this is handled by the loop structure
                    num += 1
                    if num == 1:
                        point_to_pointidx[i] = j
                    if num >= max_p:
                        # Discard point i
                        point_to_voxelidx[i] = -1
                        break
            
            if num >= max_p:
                continue
                
            point_to_voxelidx[i] = num
            
            if num == 0:
                # New voxel
                if voxel_num[s] >= max_voxels:
                    continue
                voxelidx = voxel_num[s]
                voxel_num[s] += 1
                
                coor_to_voxelidx[i] = voxelidx
                num_points_per_voxel[s, voxelidx] = 1
                
                # Store coordinates [z, y, x]
                coors[s, voxelidx, 0] = c_z
                coors[s, voxelidx, 1] = c_y
                coors[s, voxelidx, 2] = c_x
                
                # Store features
                voxels[s, voxelidx, 0, :] = points[i, :]
            else:
                # Existing voxel
                point_idx = point_to_pointidx[i]
                voxelidx = coor_to_voxelidx[point_idx]
                
                coor_to_voxelidx[i] = voxelidx
                num_points_per_voxel[s, voxelidx] += 1
                
                # Store features
                voxels[s, voxelidx, num, :] = points[i, :]

    return voxels, coors, num_points_per_voxel
```
[G] The pseudocode above simulates the deterministic CPU logic described in the notes. The actual implementation uses a CUDA kernel `dynamic_voxelize_kernel` which parallelizes the point processing but retains the $O(N^2)$ linking logic within threads or via atomic operations for the `point_to_pointidx` search, as noted in the TODO list for non-deterministic optimization [S1, unofficial]. The final output tensors are sliced to the actual number of voxels used per scale.

**Edge Cases and Pitfalls**
- **Out of Bounds**: Points outside the `point_cloud_range` are discarded. The implementation checks bounds for $x, y, z$ sequentially, setting coordinates to -1 if out of bounds [S1, unofficial].
- **Max Voxels**: If the total number of voxels exceeds `max_voxels`, subsequent points are skipped [S1, unofficial].
- **Max Points**: If a voxel exceeds `max_points_for_all_scales[s]`, additional points are discarded [S1, unofficial].
- **Determinism**: The current implementation is deterministic but inefficient ($O(N^2)$ for point linking). A non-deterministic version is planned but not yet implemented [S1, unofficial].
- **Shuffling**: Users are advised to shuffle points before calling the function because `max_voxels` may drop points, and the order of dropped points can affect the final voxel content if not shuffled [S1, unofficial].

## Operating characteristics
- **Time Complexity**: The voxel coordinate computation is $O(N)$ per scale. However, the point linking step is $O(N^2)$ in the worst case due to the nested loop searching for previous points with identical coordinates [S1, unofficial].
- **Memory Complexity**: $O(S \cdot \text{max\_voxels} \cdot P_{max} \cdot D)$ for the output tensors, plus $O(N)$ for temporary linking arrays [S1, unofficial].
- **Performance**: On an NVIDIA RTX 3090, the multiscale operator shows a 10% speed improvement over the ordinary single-scale voxelization CUDA operator [S1, unofficial].
- **Strengths**: Simplifies multi-scale feature extraction by handling multiple scales in one pass; deterministic output.
- **Limitations**: The deterministic point linking is inefficient for large point clouds; non-deterministic optimization is not yet available [S1, unofficial].
- **When to Use**: When using multi-scale LiDAR networks (e.g., MinkowskiNet, SparseConv) and deterministic voxel content is required.
- **When Not to Use**: For very large point clouds where the $O(N^2)$ linking becomes a bottleneck, or when non-deterministic speed is preferred.

## Background
Multiscale Hard Voxelization is an unofficial CUDA operator developed by Susie0909, hosted on GitHub [S1, unofficial]. It builds upon the work of normal voxelization methods, specifically those found in the MMDetection3D and MMCV libraries, which provide single-scale hard voxelization [S1, unofficial]. The operator was created to address the inconvenience of invoking single-scale voxelization multiple times for multi-scale networks. It is used in 3D object detection pipelines that utilize LiDAR point clouds, such as those based on PointPillars or VoxelNet variants [S1, unofficial].

## Sources
[S1] Susie0909/Multiscale_Voxelization_CUDA, https://github.com/Susie0909/Multiscale_Voxelization_CUDA, fetched 2026-10-27 (unofficial)
