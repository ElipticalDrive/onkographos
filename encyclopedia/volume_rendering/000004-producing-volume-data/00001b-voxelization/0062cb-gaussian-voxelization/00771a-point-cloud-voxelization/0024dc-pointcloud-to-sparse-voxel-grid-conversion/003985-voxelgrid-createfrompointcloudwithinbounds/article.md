# VoxelGrid::CreateFromPointCloudWithinBounds
## Overview
`VoxelGrid::CreateFromPointCloudWithinBounds` voxalizes a point cloud into a sparse voxel grid by mapping input points to voxel indices based on a fixed voxel size and a minimum bounding box corner, ignoring the maximum bound for coordinate calculation [S1, unofficial]. It differs from its sibling [DCX Points-to-Voxels](3783-dcx-points-to-voxels.md) and [Multi-Plane Point Cloud Voxelization and Weight Aggregation](12101-multi-plane-point-cloud-voxelization-and-weight-aggregation.md) by using a direct hash-map accumulation approach rather than structured spatial partitioning or multi-plane projections, and is significantly faster than [Pointcloud to Sparse Voxel Grid Conversion](3416-pointcloud-to-sparse-voxel-grid-conversion.md) variants that process triangle meshes [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** `PointCloud` (containing `points_` and optionally `colors_`), `voxel_size` (double), `min_bound` (`Eigen::Vector3d`), `max_bound` (`Eigen::Vector3d`) [S1, unofficial][S2, unofficial].
- **Output:** `std::shared_ptr<VoxelGrid>` containing the resulting sparse voxel structure [S1, unofficial].

**Data Structures:**
- `voxelindex_to_accpoint`: A hash map (`std::unordered_map`) mapping `Eigen::Vector3i` (voxel index) to `AvgColorVoxel` (accumulator) [S1, unofficial]. Uses `utility::hash_eigen<Eigen::Vector3i>` as the hash function [S1, unofficial].
- `AvgColorVoxel`: An accumulator object that stores the voxel index and aggregates color values. It provides methods `Add(voxel_index, color)` and `GetAverageColor()` [S1, unofficial].

**Steps:**
1. Initialize the output `VoxelGrid` [S1, unofficial].
2. Validate `voxel_size`:
   - If `voxel_size <= 0.0`, raise an error [S1, unofficial].
   - If `voxel_size * std::numeric_limits<int>::max() < (max_bound - min_bound).maxCoeff()`, raise an error indicating `voxel_size` is too small to represent the bounds within integer limits [S1, unofficial].
3. Set `output->voxel_size_ = voxel_size` and `output->origin_ = min_bound` [S1, unofficial].
4. Create an empty hash map `voxelindex_to_accpoint` [S1, unofficial].
5. Determine if the input point cloud has colors: `has_colors = input.HasColors()` [S1, unofficial].
6. Iterate through each point in the input point cloud (index $i$ from $0$ to `input.points_.size() - 1`):
   a. Skip the current point if it is outside the inclusive bounds `[min_bound, max_bound]` [S2, unofficial]. This check also skips `NaN` points because comparisons with `NaN` return false [S2, unofficial]. The condition requires all three coordinates ($x, y, z$) to be $\ge$ `min_bound` AND $\le$ `max_bound` [S2, unofficial].
   b. Calculate the reference coordinate:
      $$ \text{ref\_coord} = \frac{\text{input.points\_}[i] - \text{min\_bound}}{\text{voxel\_size}} $$
   c. Calculate the voxel index by taking the floor of each component and casting to integer:
      $$ \text{voxel\_index} = \left( \lfloor \text{ref\_coord}_x \rfloor, \lfloor \text{ref\_coord}_y \rfloor, \lfloor \text{ref\_coord}_z \rfloor \right) $$
   d. If `has_colors` is true, call `voxelindex_to_accpoint[voxel_index].Add(voxel_index, input.colors_[i])` [S1, unofficial]. Otherwise, call `voxelindex_to_accpoint[voxel_index].Add(voxel_index)` [S1, unofficial].
7. Iterate through each entry in `voxelindex_to_accpoint`:
   a. Get the `grid_index` from the accumulator (`accpoint.second.GetVoxelIndex()`) [S1, unofficial].
   b. Get the average color: if `has_colors` is true, use `accpoint.second.GetAverageColor()`; otherwise, use `Eigen::Vector3d(0, 0, 0)` [S1, unofficial].
   c. Add a new `Voxel` to the output grid using `grid_index` and the color [S1, unofficial].

**Formulas:**
- Reference coordinate transformation: $\text{ref\_coord} = (\text{input.points\_}[i] - \text{min\_bound}) / \text{voxel\_size}$, which maps world coordinates to a continuous grid coordinate system relative to `min_bound` [S1, unofficial].
- Voxel index discretization: $\text{voxel\_index} = (\lfloor \text{ref\_coord}_x \rfloor, \lfloor \text{ref\_coord}_y \rfloor, \lfloor \text{ref\_coord}_z \rfloor)$, converting continuous coordinates to integer grid indices [S1, unofficial].

**Parameters:**
- `voxel_size`: The side length of each voxel dimension (double). Must be $> 0.0$ [S1, unofficial]. Typical values depend on scene scale and required resolution [G].
- `min_bound`: The minimum corner of the bounding box (`Eigen::Vector3d`). Used as the origin for coordinate transformation [S1, unofficial].
- `max_bound`: The maximum corner of the bounding box (`Eigen::Vector3d`). Used only for error checking if `voxel_size` is too small to cover the bounds, but not used in the coordinate mapping formula [S1, unofficial].
- `input`: The input `PointCloud` containing the points to voxelize [S2, unofficial].
- Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
- The `max_bound` parameter does not influence the spatial extent or resolution of the resulting voxel grid; it is only used for error checking [S1, unofficial].
- Changing the scale of the input point cloud while keeping `voxel_size` and `min_bound` constant changes the voxel grid's spatial extent and density, which may be counterintuitive [S1, unofficial].
- Floating-point precision issues may arise when coordinates are very large or `voxel_size` is very small, though the method does not normalize by `(max_bound - min_bound)` [S1, unofficial].
- Points outside `[min_bound, max_bound]` are skipped, and `NaN` coordinates are automatically excluded due to comparison semantics [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def CreateFromPointCloudWithinBounds(input_pointcloud, voxel_size, min_bound, max_bound):
    # 1. Initialize output
    output = VoxelGrid()
    
    # 2. Validate voxel_size
    if voxel_size <= 0.0:
        raise Error("voxel_size <= 0.")
    
    # Check if voxel_size is too small to cover bounds within integer limits
    bounds_max_diff = max(max_bound.x - min_bound.x, 
                          max_bound.y - min_bound.y, 
                          max_bound.z - min_bound.z)
    if voxel_size * 2147483647 < bounds_max_diff:
        raise Error("voxel_size is too small.")
        
    output.voxel_size = voxel_size
    output.origin = min_bound
    
    # 3. Create hash map for accumulation
    voxelindex_to_accpoint = {}
    
    has_colors = input_pointcloud.has_colors()
    
    # 4. Iterate through points
    for i in range(len(input_pointcloud.points)):
        point = input_pointcloud.points[i]
        
        # Skip points outside [min_bound, max_bound] (inclusive)
        if not (point.x >= min_bound.x and point.y >= min_bound.y and point.z >= min_bound.z and
                point.x <= max_bound.x and point.y <= max_bound.y and point.z <= max_bound.z):
            continue
            
        # Calculate reference coordinate relative to min_bound
        ref_coord = (point - min_bound) / voxel_size
        
        # Calculate voxel index using floor
        voxel_index = Vector3i(
            math.floor(ref_coord.x),
            math.floor(ref_coord.y),
            math.floor(ref_coord.z)
        )
        
        # Accumulate point into voxel
        if has_colors:
            color = input_pointcloud.colors[i]
            voxelindex_to_accpoint[voxel_index].Add(voxel_index, color)
        else:
            voxelindex_to_accpoint[voxel_index].Add(voxel_index)
            
    # 5. Build output grid from accumulated voxels
    for voxel_index, accumulator in voxelindex_to_accpoint.items():
        grid_index = accumulator.GetVoxelIndex()
        
        if has_colors:
            color = accumulator.GetAverageColor()
        else:
            color = Vector3d(0, 0, 0)
            
        output.AddVoxel(Voxel(grid_index, color))
        
    return output
```

## Subtopics

### [Vertex Cluster Assignment](10352-vertex-cluster-assignment.md)
Vertex Cluster Assignment assigns a cluster ID to each vertex or point based on which voxel it occupies in a regular grid. It serves as the core assignment step within `VoxelGrid::CreateFromPointCloudWithinBounds`. Unlike methods designed for triangle meshes, it operates directly on point cloud coordinates to discretize continuous positions into a sparse voxel structure. Choose it when fast, direct point-to-voxel mapping is required without mesh traversal overhead.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of input points $N$, $O(N)$, due to a single pass through the point cloud and a single pass through the accumulated voxels [S1, unofficial].
- **Memory Complexity:** Proportional to the number of unique voxels occupied by the point cloud, due to the hash map storage [S1, unofficial].
- **Performance:** Significantly faster than `create_from_triangle_mesh_within_bounds`, which can take $>2$ minutes for voxel resolution 50, whereas this method is suitable for high resolutions like 256 [S1, unofficial].
- **Strengths:** Fast execution for point clouds; simple implementation; handles color averaging automatically [S1, unofficial].
- **Limitations:** The fixed `min_bound` origin means scaling the input point cloud changes the voxel grid's spatial coverage and density, which may require re-tuning `min_bound` or `voxel_size` for scaled data [S1, unofficial].
- **When to use:** When fast voxelization of a point cloud is required and the spatial extent is fixed relative to `min_bound` [S1, unofficial].
- **When not to use:** When the point cloud scale varies significantly and a fixed spatial resolution relative to the data's extent is required without redefining `min_bound` [S1, unofficial].

## Background
This method is part of the Open3D library's voxelization utilities [S1, unofficial]. It was discussed in the context of user issues regarding unexpected behavior when scaling point clouds, highlighting the design choice of using `min_bound` as the sole origin for coordinate transformation [S1, unofficial].

## Sources
[S1] Voxelgrid `create_from_point_cloud_within_bounds` behaviour, https://github.com/isl-org/Open3D/issues/2766, fetched 2026-10-26 (unofficial)
[S2] Skip out-of-bounds points in VoxelGrid.create_from_point_cloud_within…, https://github.com/OnlyRightNow/Open3D/commit/7ac9a1d79c8c5fc0985939a8abc08479899743a0, fetched 2026-10-29 (unofficial)
