# DCX Points-to-Voxels

Also called: Voxelization of Point Clouds

## Overview
DCX Points-to-Voxels converts a set of 3D points (from PLY files or 3D Gaussian Splatting) into a sparse voxel grid by averaging point positions and colors within grid cells defined by a bounding box and resolution [S1, S2, unofficial]. It differs from simple occupancy voxelization by storing averaged geometric and color data per voxel, and from dense voxelization by using hash maps to store only occupied cells [S1, S2, unofficial]. This method serves as the core voxelization step in pipelines converting point clouds to sparse voxel representations for downstream rendering or USD export [S1, S2, unofficial].

## How it works
**Inputs:**
- `points`: Array of shape $(N, 3)$ containing 3D coordinates of $N$ points [S1, S2, unofficial].
- `colors`: Optional array of shape $(N, 3)$ containing RGB color values [S2, unofficial].
- `resolution`: Float defining voxel size in world units (default 0.1) [S2, unofficial] or integer scaling factor [S1, unofficial].
- `density_threshold`: Integer minimum number of points required to create a voxel (default 1) [S2, unofficial].
- `color_levels`: Integer for color quantization (0=off, 8/16/32=quantize) (default 0) [S2, unofficial].

**Outputs:**
- `voxels`: A sparse map where keys are voxel IDs and values contain averaged position and/or color [S1, S2, unofficial].

**Data Structures:**
- **Auxiliary Map (`aux`)**: Hash map from `CubeID` (integer) to `count` (integer) [S1, unofficial].
- **Voxels Map (`voxels`)**: Hash map from `CubeID` (integer) to `vector<float>` of size 3 representing averaged 3D coordinates [S1, unofficial]. In S2, this is a dictionary mapping grid coordinates to objects containing `color_sum` and `count` [S2, unofficial].
- **CubeID**: Linear voxel index computed from grid coordinates [S1, unofficial].

**Steps:**

1. **Calculate Bounds and Orders:**
   - Compute bounding box extents: `min_bounds = points.min(axis=0)`, `max_bounds = points.max(axis=0)` [S2, unofficial].
   - Adjust bounds for stability:
     - `temp_0 = floor(min_bounds * resolution) - 3`
     - `temp_1 = floor(max_bounds * resolution) + 3`
     - `bbox_min = temp_0 / resolution`
     - `bbox_max = temp_1 / resolution`
     - `width[i] = temp_1[i] - temp_0[i]` for $i \in \{0, 1, 2\}$ [S1, unofficial].
   - Compute strides (orders):
     - `orders[0] = width[0]`
     - `orders[1] = width[0] * width[1]` [S1, unofficial].
   - Alternatively, compute grid size directly: `grid_size = ceil((max_bounds - min_bounds) / resolution).astype(int) + 1` [S2, unofficial].

2. **Iterate Points and Compute Grid Indices:**
   - For each point $i$ with position $p_i = (x, y, z)$:
     - Compute scaled coordinates:
       - $s_x = \lfloor (x - bbox\_min[0]) \times resolution \rfloor$
       - $s_y = \lfloor (y - bbox\_min[1]) \times resolution \rfloor$
       - $s_z = \lfloor (z - bbox\_min[2]) \times resolution \rfloor$ [S1, unofficial].
     - Alternatively, using min_bounds:
       - $grid\_coords = \lfloor (p_i - min\_bounds) / resolution \rfloor$ [S2, unofficial].

3. **Compute Voxel ID:**
   - Linearize grid coordinates to a unique ID:
     - $voxel\_id = s_x + s_y \times orders[0] + s_z \times orders[1]$ [S1, unofficial].
     - Or use tuple $(s_x, s_y, s_z)$ as key [S2, unofficial].

4. **Aggregate Data:**
   - **If using S1 (Position Averaging):**
     - Check if `voxel_id` exists in `aux`.
     - If not:
       - Insert `voxel_id` into `aux` with `count = 1`.
       - Insert `voxel_id` into `voxels` with value $p_i$.
     - If yes:
       - Retrieve existing `count` and `old_point`.
       - `new_count = count + 1`.
       - Compute merged position:
         - $merged\_pos[j] = \frac{count \times old\_point[j] + p_i[j]}{new\_count}$ for $j \in \{0, 1, 2\}$.
       - Update `voxels[voxel_id] = merged_pos`.
       - Update `aux[voxel_id] = new_count`.
   - **If using S2 (Color Accumulation):**
     - Accumulate color sums and counts:
       - `voxel_data[key]['color_sum'] += colors[i]`
       - `voxel_data[key]['count'] += 1` [S2, unofficial].

5. **Filter and Finalize:**
   - Filter voxels: If `count < density_threshold`, skip the voxel [S2, unofficial].
   - Compute final color:
     - $avg\_color = (color\_sum / count).astype(np.uint8)$ [S2, unofficial].
   - Apply color quantization if `color_levels > 0` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dcx_points_to_voxels(points, colors=None, resolution=0.1, density_threshold=1, color_levels=0):
    """
    Convert points to sparse voxels.
    points: (N, 3) array
    colors: (N, 3) array, optional
    resolution: float
    density_threshold: int
    color_levels: int
    """
    # 1. Calculate Bounds
    min_bounds = points.min(axis=0)
    max_bounds = points.max(axis=0)
    
    # Adjust bounds for stability (S1 style)
    # Note: S1 uses integer scaling for bounds adjustment, S2 uses float division.
    # We follow S2's simpler approach for general compatibility, but S1's padding is safer for edge cases.
    # Here we implement S1's padding logic for robustness.
    bbox_min = np.floor(min_bounds * resolution) - 3
    bbox_max = np.floor(max_bounds * resolution) + 3
    bbox_min = bbox_min / resolution
    bbox_max = bbox_max / resolution
    
    width = np.floor(max_bounds * resolution) + 3 - (np.floor(min_bounds * resolution) - 3)
    orders = np.zeros(2, dtype=int)
    orders[0] = int(width[0])
    orders[1] = int(width[0] * width[1])
    
    # Data structures
    aux = {}  # voxel_id -> count
    voxels = {}  # voxel_id -> [x, y, z]
    color_accum = {}  # voxel_id -> [r, g, b] sum (if colors provided)
    
    # 2. Iterate Points
    for i in range(len(points)):
        p = points[i]
        
        # Compute grid indices
        sx = int(np.floor((p[0] - bbox_min[0]) * resolution))
        sy = int(np.floor((p[1] - bbox_min[1]) * resolution))
        sz = int(np.floor((p[2] - bbox_min[2]) * resolution))
        
        # Compute Voxel ID
        voxel_id = sx + sy * orders[0] + sz * orders[1]
        
        # 3. Aggregate
        if voxel_id not in aux:
            aux[voxel_id] = 1
            voxels[voxel_id] = p.copy()
            if colors is not None:
                color_accum[voxel_id] = colors[i].copy()
        else:
            count = aux[voxel_id]
            old_pos = voxels[voxel_id]
            new_count = count + 1
            
            # Update position
            merged_pos = (count * old_pos + p) / new_count
            voxels[voxel_id] = merged_pos
            aux[voxel_id] = new_count
            
            # Update color
            if colors is not None:
                color_accum[voxel_id] += colors[i]
    
    # 4. Filter and Finalize
    result_voxels = {}
    for voxel_id, count in aux.items():
        if count < density_threshold:
            continue
        
        voxel_data = {
            'position': voxels[voxel_id],
            'count': count
        }
        
        if colors is not None and voxel_id in color_accum:
            avg_color = color_accum[voxel_id] / count
            # Quantization
            if color_levels > 0:
                # Simple quantization: scale to [0, color_levels-1] and back
                # Assuming colors are in [0, 1] or [0, 255]. S2 uses uint8.
                # Assuming input colors are [0, 255] for uint8 conversion
                avg_color = np.clip(avg_color, 0, 255).astype(np.uint8)
                # Further quantization if needed
                if color_levels < 256:
                    step = 256 / color_levels
                    avg_color = np.round(avg_color / step) * step
                    avg_color = avg_color.astype(np.uint8)
            voxel_data['color'] = avg_color
            
        result_voxels[voxel_id] = voxel_data
        
    return result_voxels
```

**Parameters:**
- `resolution`: Voxel size in world units. Default 0.1 [S2, unofficial].
- `density_threshold`: Minimum points per voxel. Default 1 [S2, unofficial].
- `color_levels`: Color quantization levels. Default 0 [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Floating point division and floor operations can cause boundary artifacts. S1 adds a padding of 3 units to the bounding box to mitigate this [S1, unofficial].
- **Empty Voxels:** Voxels with fewer points than `density_threshold` are discarded [S2, unofficial].
- **Color Quantization:** If `color_levels` is set, colors are quantized to reduce the number of unique materials, which is useful for USD PointInstancer optimization [S2, unofficial].

## Subtopics

### Greedy Meshing
Merges adjacent voxels with similar colors into larger boxes to reduce draw calls. This is a post-processing step applied after voxelization [S2, unofficial].

### USD PointInstancer
A rendering technique that instances a single geometry prototype with per-voxel transforms and colors, leveraging the sparse voxel output for efficient rendering [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of points, as each point is processed once and hash map operations are $O(1)$ on average [S1, S2, unofficial].
- **Memory Complexity:** $O(V)$ where $V$ is the number of occupied voxels. The sparse map stores only non-empty voxels [S1, S2, unofficial].
- **Performance:** Handles 1M+ voxels efficiently on GPU via instancing [S2, unofficial]. CPU voxelization is fast due to simple averaging [S1, S2, unofficial].
- **Strengths:** Simple, fast, produces sparse data suitable for large scenes [S1, S2, unofficial].
- **Limitations:** Loses high-frequency detail within voxels due to averaging [S1, unofficial]. Resolution is fixed per grid [S2, unofficial].

## Background
DCX Points-to-Voxels is part of the DCx toolkit, which converts point clouds and 3D Gaussian Splatting data into USD formats [S1, unofficial]. It is used in pipelines that require efficient rendering of large point cloud datasets by converting them into sparse voxel grids [S2, unofficial]. The method is implemented in Python and C++ for performance [S1, unofficial].

## Sources
[S1] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S2] omolism/3DGS-to-Voxel-OpenUSD, https://github.com/omolism/3DGS-to-Voxel-OpenUSD, fetched 2026-10-27 (unofficial)
