# VoxelGrid::CreateFromPointCloudWithinBounds


## Overview
A method to voxelize a point cloud within specified bounds by mapping input points to voxel indices based on a fixed voxel size and minimum bound, ignoring the maximum bound for coordinate calculation. It differs from [DCX Points-to-Voxels](3783-dcx-points-to-voxels.md) by using a hash-map accumulation strategy rather than a dense or structured sparse layout, and from [Pointcloud to Sparse Voxel Grid Conversion](3416-pointcloud-to-sparse-voxel-grid-conversion.md) by its specific reliance on `min_bound` as the sole origin for coordinate transformation.

## How it works
**Inputs and Outputs:**
- Input: `PointCloud` (points and optional colors), `voxel_size` (double), `min_bound` (Eigen::Vector3d), `max_bound` (Eigen::Vector3d).
- Output: `VoxelGrid` containing voxels with integer indices and average colors.

**Data Structures:**
- `voxelindex_to_accpoint`: A hash map (`std::unordered_map`) mapping `Eigen::Vector3i` (voxel index) to `AvgColorVoxel` (accumulator). Uses `utility::hash_eigen<Eigen::Vector3i>` as the hash function.
- `AvgColorVoxel`: An accumulator that stores the voxel index and sums color values to compute an average.

**Steps:**
1. Initialize output `VoxelGrid`.
2. Validate `voxel_size > 0.0`.
3. Validate that `voxel_size` is large enough to cover the bounds: `voxel_size * std::numeric_limits<int>::max() >= (max_bound - min_bound).maxCoeff()`. If not, log an error.
4. Set `output->voxel_size_ = voxel_size` and `output->origin_ = min_bound`.
5. Create empty `voxelindex_to_accpoint` hash map.
6. Iterate through each point in the input point cloud (index `i` from 0 to `input.points_.size()-1`):
   a. Calculate `ref_coord = (input.points_[i] - min_bound) / voxel_size`.
   b. Calculate `voxel_index` by taking the floor of each component of `ref_coord` and casting to `int`: `voxel_index << int(floor(ref_coord(0))), int(floor(ref_coord(1))), int(floor(ref_coord(2)))`.
   c. If the input point cloud has colors, call `voxelindex_to_accpoint[voxel_index].Add(voxel_index, input.colors_[i])`. Otherwise, call `voxelindex_to_accpoint[voxel_index].Add(voxel_index)`.
7. Iterate through each entry in the `voxelindex_to_accpoint` hash map.
   a. Get the `grid_index` from the accumulator (`accpoint.second.GetVoxelIndex()`).
   b. Get the average color: if `has_colors` is true, use `accpoint.second.GetAverageColor()`; otherwise, use `Eigen::Vector3d(0, 0, 0)`.
   c. Add a new `Voxel` to the output grid using the `grid_index` and color.

**Formula:**
$$ \text{ref\_coord} = \frac{\text{input.points}[i] - \text{min\_bound}}{\text{voxel\_size}} $$
$$ \text{voxel\_index} = \lfloor \text{ref\_coord} \rfloor $$

**Parameters:**
- `voxel_size`: The size of each voxel dimension (double). Must be > 0.0.
- `min_bound`: The minimum corner of the bounding box (Eigen::Vector3d). Used as the origin for coordinate transformation.
- `max_bound`: The maximum corner of the bounding box (Eigen::Vector3d). Used only for error checking if `voxel_size` is too small to cover the bounds, but not used in the coordinate mapping formula.

**Pitfalls:**
- The `max_bound` parameter is not used in the coordinate transformation formula. Changing `max_bound` does not change the voxel grid's spatial extent or resolution, only the error check for `voxel_size` validity. This causes issues when scaling the input point cloud while keeping voxel resolution constant, as the grid origin is fixed at `min_bound`.
- Floating point precision can be an issue when coordinates are large relative to `voxel_size`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CreateFromPointCloudWithinBounds(input_pointcloud, voxel_size, min_bound, max_bound):
    if voxel_size <= 0.0:
        raise Error("voxel_size <= 0.")
    
    bounds_diff = max_bound - min_bound
    max_diff = max(bounds_diff.x, bounds_diff.y, bounds_diff.z)
    if voxel_size * INT_MAX < max_diff:
        raise Error("voxel_size is too small.")
    
    output_voxelgrid = VoxelGrid()
    output_voxelgrid.voxel_size = voxel_size
    output_voxelgrid.origin = min_bound
    
    voxelindex_to_accpoint = HashMap() # Key: Vector3i, Value: AvgColorVoxel
    has_colors = input_pointcloud.has_colors()
    
    for i in range(0, input_pointcloud.points.size):
        point = input_pointcloud.points[i]
        ref_coord = (point - min_bound) / voxel_size
        
        voxel_index = Vector3i(
            floor(ref_coord.x),
            floor(ref_coord.y),
            floor(ref_coord.z)
        )
        
        if has_colors:
            color = input_pointcloud.colors[i]
            voxelindex_to_accpoint[voxel_index].Add(voxel_index, color)
        else:
            voxelindex_to_accpoint[voxel_index].Add(voxel_index)
            
    for accpoint in voxelindex_to_accpoint.values():
        grid_index = accpoint.GetVoxelIndex()
        if has_colors:
            color = accpoint.GetAverageColor()
        else:
            color = Vector3d(0, 0, 0)
        output_voxelgrid.AddVoxel(Voxel(grid_index, color))
        
    return output_voxelgrid
```

## Subtopics

### [Vertex Cluster Assignment](10352-vertex-cluster-assignment.md)
Assigns a cluster ID to each vertex/point based on which voxel it occupies in the regular grid.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of input points, $O(N)$, due to single pass iteration and hash map operations (average $O(1)$).
- **Memory Complexity:** Proportional to the number of unique voxels occupied by points, $O(V)$, where $V \le N$.
- **Performance:** Faster than `create_from_triangle_mesh_within_bounds` [S1, unofficial].
- **Strengths:** Simple implementation, efficient for sparse point clouds.
- **Limitations:** Scaling the input point cloud while keeping voxel resolution constant can lead to incorrect density or gaps because the grid origin is fixed at `min_bound` and `max_bound` is ignored for coordinate mapping [S1, unofficial].
- **When to use:** When fast voxelization of a point cloud is required and the point cloud scale is fixed relative to `min_bound`.
- **When not to use:** When the point cloud scale changes significantly relative to `min_bound` and consistent density is required, or when `max_bound` needs to influence the grid extent.

## Background
Originated in Open3D as part of the `VoxelGrid` geometry class. The method was discussed in GitHub issue #2766 regarding unexpected behavior when scaling point clouds [S1, unofficial].

## Sources
[S1] Voxelgrid `create_from_point_cloud_within_bounds` behaviour, https://github.com/isl-org/Open3D/issues/2766, fetched 2026-10-26 (unofficial)
