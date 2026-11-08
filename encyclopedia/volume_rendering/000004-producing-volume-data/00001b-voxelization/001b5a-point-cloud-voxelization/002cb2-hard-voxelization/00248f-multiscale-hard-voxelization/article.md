# Multiscale Hard Voxelization

Also called: multiscale voxelization CUDA operator

## Overview
Multiscale Hard Voxelization is a GPU-accelerated method that simultaneously voxelizes a LiDAR point cloud into multiple spatial resolutions (scales) in a single pass [S1, unofficial]. It extends standard hard voxelization by iterating over a list of voxel sizes and point limits, outputting separate voxel feature tensors, coordinates, and point counts for each scale [S1, unofficial]. It differs from single-scale voxelization by managing multiple memory blocks indexed by scale, offering a 10% speed improvement over ordinary single-scale CUDA operators on NVIDIA RTX 3090 hardware [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A point cloud tensor `points` of shape $[N, D]$, where $N$ is the number of points and $D$ is the number of features (e.g., $x, y, z, r$) [S1, unofficial].
*   **Input:** `voxel_sizes`: A list of $S$ voxel dimensions, where each scale $s$ has size $v^{(s)} = [v_x^{(s)}, v_y^{(s)}, v_z^{(s)}]$ [S1, unofficial].
*   **Input:** `max_points_for_all_scales`: A list of $S$ integers, where $M^{(s)}$ is the maximum number of points allowed per voxel for scale $s$ [S1, unofficial].
*   **Input:** `point_cloud_range`: A list of 6 floats $[x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$ defining the bounding box [S1, unofficial].
*   **Input:** `max_voxels`: The maximum total number of voxels allowed across all scales [S1, unofficial].
*   **Output:** `voxels`: Tensor of shape $[S, M_{max}, M_{max\_pts}, D]$, containing point features grouped by voxel and scale [S1, unofficial].
*   **Output:** `coors`: Tensor of shape $[S, M_{max}, 3]$, containing voxel coordinates in $[z, y, x]$ order [S1, unofficial].
*   **Output:** `num_points_per_voxel`: Tensor of shape $[S, M_{max}]$, containing the count of points in each voxel [S1, unofficial].

**Data Structures:**
The algorithm uses several auxiliary arrays allocated per scale:
1.  `point_to_voxelidx`: Array of size $N$, initialized to $-1$. Stores the voxel index assigned to each point [S1, unofficial].
2.  `point_to_pointidx`: Array of size $N$. Stores the index of the "first" point in the same voxel for each point [S1, unofficial].
3.  `coor_to_voxelidx`: Array of size $N$. Maps a point's index to its voxel index [S1, unofficial].
4.  `voxel_num`: Scalar counter tracking the number of allocated voxels for the current scale [S1, unofficial].

**Steps:**
For each scale $s$ from $0$ to $S-1$:

1.  **Initialize Grid Dimensions:**
    Calculate the grid dimensions for the current scale based on the range and voxel size:
    $$
    grid_x = \text{round}\left(\frac{x_{max} - x_{min}}{v_x^{(s)}}\right), \quad
    grid_y = \text{round}\left(\frac{y_{max} - y_{min}}{v_y^{(s)}}\right), \quad
    grid_z = \text{round}\left(\frac{z_{max} - z_{min}}{v_z^{(s)}}\right)
    $$
    [S1, unofficial]

2.  **Compute Voxel Coordinates:**
    For each point $p$ at index $i$ with coordinates $(x, y, z)$:
    $$
    c_x = \left\lfloor \frac{x - x_{min}}{v_x^{(s)}} \right\rfloor, \quad
    c_y = \left\lfloor \frac{y - y_{min}}{v_y^{(s)}} \right\rfloor, \quad
    c_z = \left\lfloor \frac{z - z_{min}}{v_z^{(s)}} \right\rfloor
    $$
    [S1, unofficial]

3.  **Boundary Check:**
    Validate coordinates against the grid dimensions $[0, grid_x)$, $[0, grid_y)$, and $[0, grid_z)$.
    *   If $c_x$ is out of bounds, set `coors[0] = -1` and skip the point [S1, unofficial].
    *   If $c_y$ is out of bounds, set `coors[0] = -1`, `coors[1] = -1` and skip [S1, unofficial].
    *   If $c_z$ is out of bounds, set `coors[0] = -1`, `coors[1] = -1`, `coors[2] = -1` and skip [S1, unofficial].

4.  **Link Points to Voxels (Deterministic):**
    To ensure deterministic ordering and handle points falling into the same voxel, the implementation uses a nested loop approach [S1, unofficial]:
    *   Initialize `num = 0`.
    *   Iterate $j$ from $0$ to $i-1$:
        *   If point $j$ has the same voxel coordinates $(c_x, c_y, c_z)$ as point $i$:
            *   Increment `num`.
            *   If `num == 1`, set `point_to_pointidx[i] = j`.
            *   If `num >= max_points_for_all_scales[s]`, discard point $i$ (set `point_to_voxelidx[i] = -1`) and break [S1, unofficial].
    *   If `num == 0`, set `point_to_pointidx[i] = i` [S1, unofficial].
    *   Set `point_to_voxelidx[i] = num` if valid, else $-1$ [S1, unofficial].

5.  **Allocate Voxel Indices:**
    *   If `point_to_voxelidx[i] == 0` (this is the first point in a new voxel):
        *   If `voxel_num[s] >= max_voxels`, skip allocation [S1, unofficial].
        *   Assign `voxelidx = voxel_num[s]`.
        *   Increment `voxel_num[s]`.
        *   Set `coor_to_voxelidx[i] = voxelidx` [S1, unofficial].
        *   Set `num_points_per_voxel[voxelidx] = 1` [S1, unofficial].
    *   If `point_to_voxelidx[i] > 0`:
        *   Find the index of the first point in the same voxel: `point_idx = point_to_pointidx[i]` [S1, unofficial].
        *   Set `coor_to_voxelidx[i] = coor_to_voxelidx[point_idx]` [S1, unofficial].
        *   Increment `num_points_per_voxel[voxelidx]` [S1, unofficial].

6.  **Store Results:**
    *   Store valid voxel coordinates in `coors` in $[z, y, x]$ order [S1, unofficial].
    *   Store point features in `voxels` up to `max_points_for_all_scales[s]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def multiscale_hard_voxelize(points, voxel_sizes, max_points_limits, point_cloud_range, max_voxels):
    """
    points: List of [x, y, z, ...] or 2D array of shape [N, D]
    voxel_sizes: List of [vx, vy, vz] for each scale
    max_points_limits: List of int for each scale
    point_cloud_range: [xmin, ymin, zmin, xmax, ymax, zmax]
    max_voxels: int
    Returns: voxels, coors, num_points_per_voxel (lists of lists per scale)
    """
    S = len(voxel_sizes)
    N = len(points)
    D = len(points[0]) if N > 0 else 0

    # Pre-allocate output structures
    max_pts_global = max(max_points_limits)
    voxels = [[[] for _ in range(max_voxels)] for _ in range(S)]
    coors = [[None for _ in range(max_voxels)] for _ in range(S)]
    num_points_per_voxel = [[0] * max_voxels for _ in range(S)]
    voxel_nums = [0] * S

    for s in range(S):
        vx, vy, vz = voxel_sizes[s]
        max_pts = max_points_limits[s]
        xmin, ymin, zmin, xmax, ymax, zmax = point_cloud_range

        # 1. Initialize Grid Dimensions
        grid_x = round((xmax - xmin) / vx)
        grid_y = round((ymax - ymin) / vy)
        grid_z = round((zmax - zmin) / vz)

        # Auxiliary arrays for this scale
        point_to_voxelidx = [-1] * N
        point_to_pointidx = [-1] * N
        coor_to_voxelidx = [-1] * N

        for i in range(N):
            x, y, z = points[i][0], points[i][1], points[i][2]

            # 2. Compute Voxel Coordinates
            c_x = math.floor((x - xmin) / vx)
            c_y = math.floor((y - ymin) / vy)
            c_z = math.floor((z - zmin) / vz)

            # 3. Boundary Check (sequential invalidation per source)
            if c_x < 0 or c_x >= grid_x:
                continue  # Marks as invalid in source logic
            if c_y < 0 or c_y >= grid_y:
                continue
            if c_z < 0 or c_z >= grid_z:
                continue

            # 4. Link Points to Voxels (Deterministic O(N^2))
            num = 0
            first_point_idx = -1
            for j in range(i):
                xj, yj, zj = points[j][0], points[j][1], points[j][2]
                cj_x = math.floor((xj - xmin) / vx)
                cj_y = math.floor((yj - ymin) / vy)
                cj_z = math.floor((zj - zmin) / vz)
                if cj_x == c_x and cj_y == c_y and cj_z == c_z:
                    num += 1
                    if num == 1:
                        first_point_idx = j
                    if num >= max_pts:
                        break

            if num >= max_pts:
                point_to_voxelidx[i] = -1
                continue

            # 5. Set point_to_voxelidx and point_to_pointidx
            point_to_voxelidx[i] = num
            if num == 0:
                point_to_pointidx[i] = i
            else:
                point_to_pointidx[i] = first_point_idx

            # 6. Allocate Voxel Indices
            if num == 0:
                if voxel_nums[s] >= max_voxels:
                    continue
                voxelidx = voxel_nums[s]
                voxel_nums[s] += 1
                coor_to_voxelidx[i] = voxelidx
                num_points_per_voxel[s][voxelidx] = 1

                # Store coors in [z, y, x] order
                coors[s][voxelidx] = [c_z, c_y, c_x]
                voxels[s][voxelidx].append(points[i])
            else:
                point_idx = point_to_pointidx[i]
                voxelidx = coor_to_voxelidx[point_idx]
                coor_to_voxelidx[i] = voxelidx
                num_points_per_voxel[s][voxelidx] += 1
                voxels[s][voxelidx].append(points[i])

    # Trim outputs to actual voxel counts per scale
    voxels_out = []
    coors_out = []
    num_points_out = []
    for s in range(S):
        count = voxel_nums[s]
        voxels_out.append([voxels[s][k][:max_points_limits[s]] for k in range(count)])
        coors_out.append([coors[s][k] for k in range(count)])
        num_points_out.append(num_points_per_voxel[s][:count])
    return voxels_out, coors_out, num_points_out
```

**Parameters:**
*   `voxel_sizes`: Nested list, e.g., `[[0.16, 0.16, 4], [0.32, 0.32, 4], [0.64, 0.64, 4]]` [S1, unofficial].
*   `max_points_for_all_scales`: List, e.g., `[32, 64, 128]` [S1, unofficial].
*   `point_cloud_range`: List of 6 floats, e.g., `[0, -39.68, -3, 69.12, 39.68, 1]` [S1, unofficial].
*   `max_voxels`: Integer, e.g., `20000` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Out of Bounds:** Points with coordinates outside the range $R$ are discarded. The implementation sets invalid coordinates to $-1$ [S1, unofficial].
*   **Determinism:** The method uses a deterministic $O(N^2)$ approach for linking points, which is inefficient for large point clouds but ensures consistent ordering [S1, unofficial]. A non-deterministic version is noted as a TODO [S1, unofficial].
*   **Max Voxels:** If `voxel_num` exceeds `max_voxels`, subsequent voxels are skipped. Users are advised to shuffle points before calling the function to mitigate bias from dropping points [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The deterministic linking step is $O(N^2)$ per scale due to the nested loop checking previous points [S1, unofficial]. This is a significant limitation for large point clouds.
*   **Memory Complexity:** $O(S \cdot M_{max} \cdot M_{max\_pts} \cdot D)$ for the output tensors, where $S$ is the number of scales, $M_{max}$ is `max_voxels`, and $M_{max\_pts}$ is the maximum `max_points` across all scales [S1, unofficial].
*   **Performance:** On an NVIDIA RTX 3090, the multiscale operator shows a 10% speed improvement over the ordinary single-scale voxelization CUDA operator [S1, unofficial].
*   **Strengths:** Simplifies API by handling multiple scales in one call; faster than running single-scale voxelization multiple times [S1, unofficial].
*   **Limitations:** The deterministic linking is inefficient; not batched; requires `deterministic=True` for the current stable version [S1, unofficial].

## Background
Multiscale Hard Voxelization was introduced as a CUDA operator to address the inconvenience of invoking single-scale voxelization multiple times for feature extraction in 3D object detection networks [S1, unofficial]. It builds upon the normal voxelization method, specifically those found in the MMDetection3D ecosystem [S1, unofficial]. The implementation is based on the `mmcv` and `mmdet` open-source codebases [S1, unofficial]. It is commonly used in LiDAR-based 3D perception tasks where multi-scale feature representation is beneficial for detecting objects of varying sizes [S1, unofficial].

## Sources
[S1] Susie0909/Multiscale_Voxelization_CUDA, https://github.com/Susie0909/Multiscale_Voxelization_CUDA, fetched 2026-10-27 (unofficial)
