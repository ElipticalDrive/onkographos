# Point Cloud Voxelization

Also called: Pointcloud to Dense Voxel Grid Conversion, Voxelization from Point Cloud

## Overview
Point Cloud Voxelization converts an unstructured set of 3D points into a discrete volumetric grid representation suitable for rendering or spatial queries [S2][S5, unofficial]. It solves the problem of mapping irregular point data into a regular structure by binning points into cubic cells based on spatial coordinates [S3, S5, unofficial]. It differs from [Voxel Grid from Mesh (Fill Method)](1013-voxelgrid-from-mesh-fill-method.md) by operating on discrete samples rather than continuous surfaces, and from [Hard Voxelization](5345-hard-voxelization.md) by often supporting feature aggregation (e.g., averaging colors) rather than simple occupancy [S2][S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A point cloud array `points` of shape $(N, D)$, where the first 3 columns are $(x, y, z)$ coordinates and the remaining $D-3$ columns are features (e.g., color, intensity) [S3, S5, unofficial].
*   **Parameters:**
    *   `voxel_size`: A 3D vector $(s_x, s_y, s_z)$ defining the physical dimensions of each voxel [S3, S4, unofficial].
    *   `grid_range` or `start_location`/`end_location`: The bounding box of the output grid. `grid_range` is typically $(min_x, min_y, min_z, max_x, max_y, max_z)$ [S3, unofficial]. `start_location` and `end_location` define the min and max bounds explicitly [S5, unofficial].
    *   `max_points_in_voxel`: Integer limit on points stored per voxel (default 60) [S3, unofficial].
    *   `max_num_voxels`: Integer limit on total allocated voxels (default 20000) [S3, unofficial].
    *   `segment_func`: Aggregation function for features (e.g., mean, sum, min, max) [S5, unofficial].
*   **Output:**
    *   **Dense Grid Variant:** A tensor of shape $(G_x, G_y, G_z, F)$ containing aggregated voxel features [S5, unofficial].
    *   **Sparse Grid Variant:** Three arrays: `voxels` (shape $(V, M, D)$), `coordinates` (shape $(V, 3)$), and `num_points_per_voxel` (shape $(V,)$), where $V$ is the number of occupied voxels and $M$ is `max_points_in_voxel` [S3, unofficial].

**Data Structures:**
1.  **Coordinate Map (`coor_to_voxelidx`):** A 3D array of shape $(G_z, G_y, G_x)$ with dtype `int32`, initialized to $-1$. It maps grid indices to a sparse linear voxel index [S3, unofficial].
2.  **Voxel Buffer (`voxels`):** A 3D array of shape $(max\_num\_voxels, max\_points\_in\_voxel, D)$ storing raw point data for each occupied voxel [S3, unofficial].
3.  **Coordinate Buffer (`coors`):** A 2D array of shape $(max\_num\_voxels, 3)$ storing the grid coordinates $(i, j, k)$ of each occupied voxel [S3, unofficial].
4.  **Point Count Buffer (`num_points_per_voxel`):** A 1D array of shape $(max\_num\_voxels)$ storing the number of points currently stored in each voxel [S3, unofficial].

**Steps:**

1.  **Grid Dimension Calculation:**
    Compute the grid size $(G_x, G_y, G_z)$ based on the bounding box and voxel size.
    $$G = \left\lfloor \frac{grid\_range[3:] - grid\_range[:3]}{voxel\_size} \right\rfloor$$
    Cast $G$ to `int32` [S3, unofficial]. Alternatively, using start/end locations:
    $$G = \left\lceil \frac{end\_location - start\_location}{grid\_cell\_size} \right\rceil$$
    Cast $G$ to `int32` [S5, unofficial].

2.  **Coordinate Discretization:**
    Convert continuous point coordinates to grid indices.
    $$p_{grid} = \left\lfloor \frac{points[:, :3] - grid\_range[:3]}{voxel\_size} \right\rfloor$$
    Cast to `int32` [S3, unofficial].
    *Note:* Some implementations use 1-based indexing or add offsets depending on the language (e.g., Julia) [S4, unofficial].

3.  **Boundary Masking and Clamping:**
    *   **Discard Method:** Create a boolean mask for points where $0 \le p_{grid} < G$ in all dimensions. Discard points outside this range [S3, unofficial].
    *   **Clamp Method:** Clamp indices to $[0, G-1]$ for points outside the bounds, assigning them to the nearest edge voxel [S5, unofficial].

4.  **Sparse Index Mapping (Sparse Variant Only):**
    Initialize `coor_to_voxelidx` to $-1$. Iterate through valid points. For each point with grid coordinate $c$:
    *   Look up `voxel_id = coor_to_voxelidx[c]`.
    *   If `voxel_id == -1`:
        *   Assign `voxel_id = voxel_num`.
        *   Increment `voxel_num`.
        *   If `voxel_num > max_num_voxels`, break processing [S3, unofficial].
        *   Set `coor_to_voxelidx[c] = voxel_id`.
        *   Store $c$ in `coors[voxel_id]`.
    *   Store the point data in `voxels[voxel_id, num_points_per_voxel[voxel_id]]`.
    *   Increment `num_points_per_voxel[voxel_id]`.
    *   *Pitfall:* If a voxel exceeds `max_points_in_voxel`, subsequent points in that voxel are discarded [S3, unofficial].

5.  **Feature Aggregation (Dense Variant Only):**
    Compute a flattened segment ID for each point:
    $$segment\_id = \sum_{d=0}^{2} p_{grid}[d] \times \prod_{k=d+1}^{2} G[k]$$
    Apply `segment_func` (e.g., mean) to aggregate features from `points` into `voxel_features` of shape $(G_x, G_y, G_z, F)$ [S5, unofficial].

**Pseudocode (Sparse Variant):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def point_cloud_voxelization_sparse(points, voxel_size, grid_range, max_points_in_voxel=60, max_num_voxels=20000):
    """
    Converts a point cloud to a sparse voxel grid.
    points: (N, D) array, first 3 cols are xyz.
    voxel_size: (3,) array.
    grid_range: (6,) array [min_x, min_y, min_z, max_x, max_y, max_z].
    """
    # 1. Grid Size
    grid_size = np.floor((grid_range[3:] - grid_range[:3]) / voxel_size).astype(np.int32)
    
    # 2. Initialize Structures
    # coor_to_voxelidx shape: (Gz, Gy, Gx)
    coor_to_voxelidx = np.full((grid_size[2], grid_size[1], grid_size[0]), -1, dtype=np.int32)
    voxels = np.zeros((max_num_voxels, max_points_in_voxel, points.shape[1]), dtype=points.dtype)
    coors = np.zeros((max_num_voxels, 3), dtype=np.int32)
    num_points_per_voxel = np.zeros(max_num_voxels, dtype=np.int32)
    
    # 3. Discretize Coordinates
    points_copy = points.copy()
    coor = np.floor((points_copy[:, :3] - grid_range[:3]) / voxel_size).astype(np.int32)
    
    # 4. Mask Valid Points
    mask = (
        (coor[:, 0] >= 0) & (coor[:, 0] < grid_size[0]) &
        (coor[:, 1] >= 0) & (coor[:, 1] < grid_size[1]) &
        (coor[:, 2] >= 0) & (coor[:, 2] < grid_size[2])
    )
    coor = coor[mask]
    points_copy = points_copy[mask]
    
    # 5. Map to Sparse Indices
    voxel_num = 0
    for i, c in enumerate(coor):
        # c is (x, y, z). Note: Some implementations reverse axis order here.
        # Assuming standard numpy indexing (x, y, z) -> (0, 1, 2)
        # However, source [S3, unofficial] uses coor[mask, ::-1] which reverses to (z, y, x)
        # to match memory layout or specific API requirements. 
        # We will follow the source logic of reversing axes for the lookup key if necessary,
        # but here we assume c is already aligned or we adjust lookup.
        # Let's stick to the source [S3, unofficial] explicit reversal:
        c_rev = c[::-1] 
        
        voxel_id = coor_to_voxelidx[c_rev[0], c_rev[1], c_rev[2]]
        
        if voxel_id == -1:
            voxel_id = voxel_num
            voxel_num += 1
            if voxel_num > max_num_voxels:
                break
            coor_to_voxelidx[c_rev[0], c_rev[1], c_rev[2]] = voxel_id
            coors[voxel_id] = c # Store original order or reversed? Source stores c (original) in coors? 
                                 # Source: coors[voxel_id] = c. But c was reversed in the loop variable?
                                 # In source: c comes from coor[mask, ::-1]. So c is reversed.
                                 # So coors stores reversed coordinates.
        
        n_pts = num_points_per_voxel[voxel_id]
        if n_pts < max_points_in_voxel:
            voxels[voxel_id, n_pts] = points_copy[i]
            num_points_per_voxel[voxel_id] += 1
            
    return voxels[:voxel_num], coors[:voxel_num], num_points_per_voxel[:voxel_num]
```

## Subtopics

### [Octree Construction for Point Clouds](1070-octree-construction-for-point-clouds.md)
Constructs a hierarchical octree for variable-resolution storage, chosen when point density varies significantly.

### [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md)
Reduces point density while preserving topological structure, used for compression before voxelization.

### [Voxels to Points Projection](3172-voxels-to-points-projection.md)
Reverses the process, mapping voxel features back to original points, used for feature propagation.

### [Pointcloud to Sparse Voxel Grid Conversion](3416-pointcloud-to-sparse-voxel-grid-conversion.md)
A variant focusing on sparse storage formats for memory efficiency in large scenes.

### [Ray Model Voxelization](4688-ray-model-voxelization.md)
Uses ray casting from sensor origins to classify voxels as free/occupied, handling LiDAR data specifically.

### [Hard Voxelization](5345-hard-voxelization.md)
Strict binning without feature aggregation, used for simple occupancy grids.

### [Gaussian Voxelization](8884-gaussian-voxelization.md)
Converts Gaussian primitives to voxels using soft kernels, for differentiable rendering.

## Operating characteristics
*   **Time Complexity:** $O(N)$ for coordinate discretization and masking. The sparse mapping loop is $O(N)$ in the best case but depends on hash/map lookups or array indexing. In dense aggregation, it is $O(N)$ plus the cost of the aggregation function [S3, S5, unofficial].
*   **Memory Complexity:**
    *   Sparse: $O(V \cdot M)$, where $V$ is occupied voxels and $M$ is `max_points_in_voxel`. The coordinate map is $O(G_x G_y G_z)$ [S3, unofficial].
    *   Dense: $O(G_x G_y G_z \cdot F)$ [S5, unofficial].
*   **Performance:** JIT-compiled versions (e.g., Numba) significantly reduce Python overhead, offering near-C performance [S3, unofficial].
*   **Strengths:** Fast, deterministic, suitable for CNN-based 3D learning [S5, unofficial].
*   **Limitations:** Fixed resolution limits detail; large bounding boxes with sparse points waste memory in dense grids; boundary handling (discard vs. clamp) affects edge accuracy [S3, S5, unofficial].

## Background
Point cloud voxelization is a fundamental preprocessing step in 3D computer vision and graphics. It bridges the gap between unstructured sensor data (LiDAR, photogrammetry) and structured grid-based algorithms. Open3D provides a standard implementation for creating voxel grids from point clouds, averaging colors for occupied voxels [S2]. TensorFlow-based implementations (e.g., TF3D) support differentiable aggregation for neural network training [S5, unofficial]. Sparse variants are widely used in autonomous driving perception pipelines (e.g., VoxelNet) to handle large-scale outdoor scenes efficiently [S3, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S3] danielhavir/voxelize3d, https://github.com/danielhavir/voxelize3d, fetched 2026-10-27 (unofficial)
[S4] sgffoen/PointCloudVoxelization, https://github.com/sgffoen/PointCloudVoxelization, fetched 2026-10-27 (unofficial)
[S5] https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py#L305, https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py, fetched 2026-10-27 (unofficial)
