# DCX Points-to-Voxels

Also called: Custom Voxel Grid Implementation, Hair Point Voxelization, Point Cloud Voxelization via Space Index Array, Voxelization of Point Clouds

## Overview
DCX Points-to-Voxels converts a set of 3D points (from PLY files or 3D Gaussian Splatting) into a sparse voxel grid by averaging point positions and colors within grid cells defined by a bounding box and resolution [S1, S2, S3, unofficial]. It differs from simple occupancy voxelization by storing averaged geometric and color data per voxel, and from dense voxelization by using hash maps or sparse arrays to store only occupied cells [S1, S2, S3, unofficial]. This method serves as the core voxelization step in pipelines converting point clouds to sparse voxel representations for downstream rendering or USD export [S1, S2, S3, unofficial].

## How it works
**Inputs:**
- `points`: Array of shape $(N, 3)$ containing 3D coordinates of $N$ points [S1, S2, S3, unofficial].
- `colors`: Optional array of shape $(N, 3)$ containing RGB color values [S2, S3, unofficial].
- `resolution`: Float defining voxel size in world units (default 0.1) [S2, S3, unofficial] or integer scaling factor [S1, unofficial].
- `density_threshold`: Integer minimum number of points required to create a voxel (default 1) [S2, S3, unofficial].
- `color_levels`: Integer for color quantization (0=off, 8/16/32=quantize) (default 0) [S2, S3, unofficial].
- `bbox_mode`: String, either "AABB" (computed from data) or "CUSTOM" (fixed dimensions) [S2, unofficial].

**Outputs:**
- `voxels`: A sparse map where keys are voxel IDs and values contain averaged position, color sum, and point count [S1, S2, S3, unofficial].

**Data Structures:**
- **Auxiliary Map (`aux`)**: Hash map from `voxel_id` (integer) to `count` (integer) [S1, unofficial].
- **Voxels Map (`voxels`)**: Hash map from `voxel_id` to a structure containing averaged 3D coordinates and accumulated color sums [S1, S2, S3, unofficial].
- **CubeID / Voxel ID**: Linear voxel index computed from grid coordinates [S1, S2, unofficial].

**Steps:**
1. **Calculate Bounds and Orders:**
   - Compute bounding box extents: `min_bounds = points.min(axis=0)`, `max_bounds = points.max(axis=0)` [S2, S3, unofficial].
   - Adjust bounds for stability (S1 style):
     - `temp_0[i] = floor(min_bounds[i] * resolution) - 3` [S1, unofficial].
     - `temp_1[i] = floor(max_bounds[i] * resolution) + 3` [S1, unofficial].
     - `bbox_min[i] = temp_0[i] / resolution`
     - `bbox_max[i] = temp_1[i] / resolution`
     - `width[i] = temp_1[i] - temp_0[i]` for $i \in \{0, 1, 2\}$ [S1, unofficial].
   - Compute strides (orders):
     - `orders[0] = width[0]`
     - `orders[1] = width[0] * width[1]` [S1, unofficial].
   - Alternatively, compute grid size directly: `grid_size = ceil((max_bounds - min_bounds) / resolution).astype(int) + 1` [S2, S3, unofficial].

2. **Iterate Points and Compute Grid Indices:**
   - For each point $i$ with position $p_i = (x, y, z)$:
     - Compute scaled coordinates:
       - $s_x = \lfloor (x - bbox\_min[0]) \times resolution \rfloor$ [S1, unofficial].
       - $s_y = \lfloor (y - bbox\_min[1]) \times resolution \rfloor$ [S1, unofficial].
       - $s_z = \lfloor (z - bbox\_min[2]) \times resolution \rfloor$ [S1, unofficial].
     - Alternatively, using min_bounds:
       - $grid\_coords = \lfloor (p_i - min\_bounds) / resolution \rfloor$ [S2, S3, unofficial].

3. **Compute Voxel ID:**
   - Linearize grid coordinates to a unique ID:
     - $voxel\_id = s_x + s_y \times orders[0] + s_z \times orders[1]$ [S1, unofficial].
     - Or use tuple $(s_x, s_y, s_z)$ as key [S2, S3, unofficial].

4. **Aggregate Data:**
   - **If using S1 (Position Averaging):**
     - Check if `voxel_id` exists in `aux`.
     - If not:
       - Insert `voxel_id` into `aux` with `count = 1` [S1, unofficial].
       - Insert `voxel_id` into `voxels` with value $p_i$ [S1, unofficial].
     - If yes:
       - Retrieve existing `count` and `old_point` [S1, unofficial].
       - `new_count = count + 1`.
       - Compute merged position:
         - $merged\_pos[j] = \frac{count \times old\_point[j] + p_i[j]}{new\_count}$ for $j \in \{0, 1, 2\}$ [S1, unofficial].
       - Update `voxels[voxel_id] = merged_pos` [S1, unofficial].
       - Update `aux[voxel_id] = new_count`.
   - **If using S2/S3 (Color Accumulation):**
     - Accumulate color sums and counts:
       - `voxel_data[key]['color_sum'] += colors[i]` [S3, unofficial].
       - `voxel_data[key]['count'] += 1` [S3, unofficial].

5. **Filter and Finalize:**
   - Filter voxels: If `count < density_threshold`, skip the voxel [S2, S3, unofficial].
   - Compute final color:
     - $avg\_color = (color\_sum / count).astype(uint8)$ [S2, S3, unofficial].
   - Apply color quantization if `color_levels > 0` [S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dcx_points_to_voxels(points, colors=None, resolution=0.1, density_threshold=1, color_levels=0):
    """
    Convert points to sparse voxels.
    points: (N, 3) array-like
    colors: (N, 3) array-like, optional
    resolution: float
    density_threshold: int
    color_levels: int
    """
    # 1. Calculate Bounds and Orders (S1 style padding)
    min_bounds = [min(p[j] for p in points) for j in range(3)]
    max_bounds = [max(p[j] for p in points) for j in range(3)]
    
    temp_0 = [math.floor(min_bounds[j] * resolution) - 3 for j in range(3)]
    temp_1 = [math.floor(max_bounds[j] * resolution) + 3 for j in range(3)]
    
    bbox_min = [temp_0[j] / resolution for j in range(3)]
    bbox_max = [temp_1[j] / resolution for j in range(3)]
    
    width = [temp_1[j] - temp_0[j] for j in range(3)]
    orders = [width[0], width[0] * width[1]]
    
    aux = {}
    voxels = {}
    color_accum = {}
    
    # 2. Iterate Points and Compute Grid Indices
    for i in range(len(points)):
        p = points[i]
        
        sx = math.floor((p[0] - bbox_min[0]) * resolution)
        sy = math.floor((p[1] - bbox_min[1]) * resolution)
        sz = math.floor((p[2] - bbox_min[2]) * resolution)
        
        voxel_id = sx + sy * orders[0] + sz * orders[1]
        
        # 3. Aggregate Data
        if voxel_id not in aux:
            aux[voxel_id] = 1
            voxels[voxel_id] = [p[0], p[1], p[2]]
            if colors is not None:
                color_accum[voxel_id] = [colors[i][0], colors[i][1], colors[i][2]]
        else:
            count = aux[voxel_id]
            old_pos = voxels[voxel_id]
            new_count = count + 1
            
            merged_pos = [(count * old_pos[j] + p[j]) / new_count for j in range(3)]
            voxels[voxel_id] = merged_pos
            aux[voxel_id] = new_count
            
            if colors is not None:
                color_accum[voxel_id] = [
                    color_accum[voxel_id][0] + colors[i][0],
                    color_accum[voxel_id][1] + colors[i][1],
                    color_accum[voxel_id][2] + colors[i][2]
                ]
    
    # 4. Filter and Finalize
    result_voxels = {}
    for voxel_id, count in aux.items():
        if count < density_threshold:
            continue
        
        voxel_data = {'position': voxels[voxel_id], 'count': count}
        
        if colors is not None and voxel_id in color_accum:
            avg_color = [color_accum[voxel_id][k] / count for k in range(3)]
            if color_levels > 0:
                step = 256.0 / color_levels
                avg_color = [max(0, min(255, int(round(c / step) * step))) for c in avg_color]
            voxel_data['color'] = avg_color
            
        result_voxels[voxel_id] = voxel_data
        
    return result_voxels
```

**Parameters:**
- `resolution`: Voxel size in world units. Default 0.1 [S2, S3, unofficial].
- `density_threshold`: Minimum points per voxel. Default 1 [S2, S3, unofficial].
- `color_levels`: Color quantization levels. Default 0 [S2, S3, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating point division and floor operations can cause boundary artifacts. S1 adds a padding of 3 units to the bounding box to mitigate this [S1, unofficial].
- **Empty Voxels:** Voxels with fewer points than `density_threshold` are discarded [S2, S3, unofficial].
- **Color Quantization:** If `color_levels` is set, colors are quantized to reduce the number of unique materials, which is useful for USD PointInstancer optimization [S2, S3, unofficial].
- **Index Mapping Bug (S2):** The original S2 implementation uses a fixed 3D array with mapping `space_Index[ijk1][ijk0-1][ijk2]` [S2, unofficial]. The `-1` offset on the X-index causes an out-of-bounds access when `ijk0 == 0` [S2, unofficial]. The hash-map approach in the pseudocode avoids this bug [G]. Additionally, S2's "CUSTOM" bounding box uses fixed dimensions that can cause out-of-bounds indices if points fall outside the range [S2, unofficial].

## Subtopics

### Greedy Meshing
Merges adjacent voxels with similar colors into larger boxes to reduce draw calls [S3, unofficial]. It sets itself apart from raw voxelization by reducing geometry count, and is chosen when optimizing for USD PointInstancer draw calls [S3, unofficial]. See [Greedy Meshing](greedy-meshing.md).

### USD PointInstancer
A rendering technique that instances a single geometry prototype with per-voxel transforms and colors, leveraging the sparse voxel output for efficient rendering [S3, unofficial]. It sets itself apart from individual mesh rendering by drastically reducing draw calls, and is chosen when exporting to USD or rendering large voxel sets [S3, unofficial]. See [USD PointInstancer](usd-point-instancer.md).

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of points, as each point is processed once and hash map operations are $O(1)$ on average [S1, S2, S3, unofficial].
- **Memory Complexity:** $O(V)$ where $V$ is the number of occupied voxels. The sparse map stores only non-empty voxels [S1, S2, S3, unofficial]. (Note: S2 uses a fixed $192 \times 128 \times 128$ array, which is $O(W \times H \times D)$ [S2, unofficial]).
- **Performance:** Handles 1M+ voxels efficiently on GPU via instancing [S3, unofficial]. CPU voxelization is fast due to simple averaging [S1, S2, unofficial].
- **Strengths:** Simple, fast, produces sparse data suitable for large scenes [S1, S2, S3, unofficial].
- **Limitations:** Loses high-frequency detail within voxels due to averaging [S1, unofficial]. Resolution is fixed per grid [S2, S3, unofficial].

## Background
DCX Points-to-Voxels is part of the DCx toolkit, which converts point clouds and 3D Gaussian Splatting data into USD formats [S1, S3, unofficial]. It is used in pipelines that require efficient rendering of large point cloud datasets by converting them into sparse voxel grids [S2, S3, unofficial]. The method is implemented in Python and C++ for performance [S1, S3, unofficial].

## Sources
[S1] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S2] ericmaic/Point-Cloud-Voxelization, https://github.com/ericmaic/Point-Cloud-Voxelization, fetched 2026-10-27 (unofficial)
[S3] omolism/3DGS-to-Voxel-OpenUSD, https://github.com/omolism/3DGS-to-Voxel-OpenUSD, fetched 2026-10-27 (unofficial)
