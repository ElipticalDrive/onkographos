# Point Cloud Voxelization

Also called: Pointcloud to Dense Voxel Grid Conversion, Voxelization from Point Cloud

## Overview
Point Cloud Voxelization converts an unstructured set of 3D points into a discrete volumetric grid representation suitable for rendering or processing [S2][S5, unofficial]. It solves the problem of mapping continuous spatial coordinates to a fixed-size array by assigning each point to a specific voxel cell based on its position relative to a defined bounding box and voxel size [S3, unofficial]. Unlike [Hard Voxelization](5345-hard-voxelization.md), which typically focuses on simple occupancy, this method often aggregates point features (such as color or intensity) into the voxel cells, providing a dense or sparse structured output [S2][S5, unofficial]. It differs from [Ray Model Voxelization](4688-ray-model-voxelization.md) by not explicitly modeling free space between the sensor and points, and from [Voxels to Points Projection](3172-voxels-to-points-projection.md) by aggregating data into the grid rather than projecting it back [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A point cloud array `points` of shape $(N, D)$, where $N$ is the number of points and $D \ge 3$ (first 3 dimensions are $x, y, z$ coordinates) [S3, S5, unofficial]. Optional associated features `features` of shape $(N, F)$ [S5, unofficial].
*   **Parameters:**
    *   `voxel_size`: Array of shape $(3,)$ specifying the physical size of each voxel in $x, y, z$ dimensions [S3, S5, unofficial].
    *   `grid_range` or `start_location`/`end_location`: Defines the spatial bounding box. `grid_range` is shape $(6,)$ as $[min_x, min_y, min_z, max_x, max_y, max_z]$ [S3, unofficial]. Alternatively, `start_location` (shape $(3,)$) and `end_location` (shape $(3,)$) define the bounds explicitly [S5, unofficial].
    *   `max_num_voxels`: Integer limit on the total number of voxels in the output grid (for sparse formats) [S3, unofficial].
    *   `max_points_in_voxel`: Integer limit on points stored per voxel (for sparse formats) [S3, unofficial].
    *   `segment_func`: Aggregation function for features (e.g., mean, sum, min, max) [S5, unofficial].
*   **Output:**
    *   **Dense:** A 4D tensor of shape $(G_x, G_y, G_z, F)$ containing aggregated voxel features [S5, unofficial].
    *   **Sparse:** A tuple `(voxels, coordinates, num_points_per_voxel)` where `voxels` is shape $(V, M, D)$, `coordinates` is shape $(V, 3)$, and `num_points_per_voxel` is shape $(V,)$, with $V \le max\_num\_voxels$ and $M \le max\_points\_in\_voxel$ [S3, unofficial].

**Data Structures:**
1.  **Grid Size:** Computed as $G = \lceil (B_{max} - B_{min}) / S \rceil$ or $\lfloor (B_{max} - B_{min}) / S \rfloor$ depending on implementation [S3, S5, unofficial].
2.  **Coordinate Mapping:** A 3D array `coor_to_voxelidx` of shape $(G_z, G_y, G_x)$ initialized to $-1$, mapping grid indices to sparse voxel indices [S3, unofficial].
3.  **Voxel Storage:** A 3D array `voxels` of shape $(max\_num\_voxels, max\_points\_in\_voxel, D)$ storing point data [S3, unofficial].

**Steps:**
1.  **Normalization/Rescaling (Optional):** Some implementations rescale points to a local $[-0.5, 0.5]$ range using a center `loc` and scale `scale`: $p' = (p - loc) / scale$ [S1, unofficial].
2.  **Grid Dimension Calculation:** Compute the number of voxels in each dimension:
    $$ G_x = \lfloor (max_x - min_x) / s_x \rfloor, \quad G_y = \lfloor (max_y - min_y) / s_y \rfloor, \quad G_z = \lfloor (max_z - min_z) / s_z \rfloor $$
    [S3, S4, unofficial]. Alternatively, using ceiling for grid size calculation [S5, unofficial].
3.  **Coordinate Discretization:** For each point $p_i = (x_i, y_i, z_i)$, compute its grid index $(i, j, k)$:
    $$ i = \lfloor (x_i - min_x) / s_x \rfloor, \quad j = \lfloor (y_i - min_y) / s_y \rfloor, \quad k = \lfloor (z_i - min_z) / s_z \rfloor $$
    [S3, S4, S5, unofficial]. Note: Some implementations use 1-based indexing or add offsets [S4, unofficial].
4.  **Boundary Handling:**
    *   **Discard:** Points with indices outside $[0, G_x-1] \times [0, G_y-1] \times [0, G_z-1]$ are discarded via a boolean mask [S3, unofficial].
    *   **Clamp:** Points outside bounds are clamped to the nearest edge voxel (index 0 or $G-1$) [S5, unofficial].
5.  **Feature Aggregation (Dense Mode):**
    *   Compute a 1D segment ID for each point: $seg_i = i \cdot (G_y G_z) + j \cdot G_z + k$ [S5, unofficial].
    *   Apply `segment_func` (e.g., mean) to aggregate `features` into the voxel grid [S5, unofficial].
6.  **Sparse Storage (Sparse Mode):**
    *   Initialize `coor_to_voxelidx` with $-1$.
    *   Iterate through valid points. For point $p$ with grid coords $c=(i,j,k)$:
        *   Look up `voxel_id = coor_to_voxelidx[c]`.
        *   If `voxel_id == -1`: Assign `voxel_id = current_voxel_count`. Increment `current_voxel_count`. If `current_voxel_count > max_num_voxels`, stop. Store `c` in `coordinates[voxel_id]`. Update `coor_to_voxelidx[c] = voxel_id`.
        *   Store point data in `voxels[voxel_id, num_points_per_voxel[voxel_id]]`.
        *   Increment `num_points_per_voxel[voxel_id]`. If count exceeds `max_points_in_voxel`, discard subsequent points in that voxel [S3, unofficial].

**Pseudocode (Sparse Implementation):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def point_cloud_voxelization_sparse(points, voxel_size, grid_range, max_points_in_voxel=60, max_num_voxels=20000):
    """
    points: (N, D) array, first 3 cols are x,y,z
    voxel_size: (3,) array [sx, sy, sz]
    grid_range: (6,) array [min_x, min_y, min_z, max_x, max_y, max_z]
    """
    # 1. Compute grid size
    grid_size = np.floor((grid_range[3:] - grid_range[:3]) / voxel_size).astype(np.int32)
    
    # 2. Initialize data structures
    # coor_to_voxelidx maps (z, y, x) to sparse index. Note axis order often reversed in implementation
    coor_to_voxelidx = np.full((grid_size[2], grid_size[1], grid_size[0]), -1, dtype=np.int32)
    voxels = np.zeros((max_num_voxels, max_points_in_voxel, points.shape[1]), dtype=points.dtype)
    coordinates = np.zeros((max_num_voxels, 3), dtype=np.int32)
    num_points_per_voxel = np.zeros(max_num_voxels, dtype=np.int32)
    
    # 3. Compute initial coordinates
    # Subtract min, divide by voxel size, floor
    coor = np.floor((points[:, :3] - grid_range[:3]) / voxel_size).astype(np.int32)
    
    # 4. Mask points within bounds
    # Check 0 <= coor < grid_size for all dimensions
    mask = (
        (coor[:, 0] >= 0) & (coor[:, 0] < grid_size[0]) &
        (coor[:, 1] >= 0) & (coor[:, 1] < grid_size[1]) &
        (coor[:, 2] >= 0) & (coor[:, 2] < grid_size[2])
    )
    coor = coor[mask]
    points = points[mask]
    
    # 5. Iterate and fill
    voxel_num = 0
    for i, c in enumerate(coor):
        # Note: Implementation may reverse axis order (c[::-1]) for memory layout
        voxel_id = coor_to_voxelidx[c[2], c[1], c[0]]
        
        if voxel_id == -1:
            voxel_id = voxel_num
            voxel_num += 1
            if voxel_num > max_num_voxels:
                break
            coor_to_voxelidx[c[2], c[1], c[0]] = voxel_id
            coordinates[voxel_id] = c
            
        n_pts = num_points_per_voxel[voxel_id]
        if n_pts < max_points_in_voxel:
            voxels[voxel_id, n_pts] = points[i]
            num_points_per_voxel[voxel_id] += 1
            
    return voxels[:voxel_num], coordinates[:voxel_num], num_points_per_voxel[:voxel_num]
```

## Subtopics

### [Octree Construction for Point Clouds](1070-octree-construction-for-point-clouds.md)
Constructs a hierarchical spatial data structure (octree) to organize point cloud objects, recursively subdividing space into eight octants until a leaf condition is met. It differs from flat voxel grids by maintaining a list of points at each node and aggregating their average position and color for spatial queries.

### [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md)
Resamples a point cloud based on active and neighbor status to support partial, percentage, and full resampling strategies. It controls point density by selectively activating points tied to tetrahedral connectivity.

### [Voxels to Points Projection](3172-voxels-to-points-projection.md)
Maps features stored in a dense voxel grid back to the original points in a point cloud using pre-computed segment IDs or point-to-voxel mappings. It solves the problem of recovering per-point features after they have been aggregated into voxels, enabling downstream tasks like point-wise classification or regression. Unlike [Hard Voxelization](5345-hard-voxelization.md) which aggregates points into voxels, this method reverses the flow, projecting voxel data back to points. It differs from [Point Cloud Cropping and Downsampling](7057-point-cloud-cropping-and-downsampling.md) by preserving the full feature dimensionality and spatial correspondence rather than reducing point count.

### [Pointcloud to Sparse Voxel Grid Conversion](3416-pointcloud-to-sparse-voxel-grid-conversion.md)
Maps a set of 3D points into a sparse grid of voxel centers and associated features, retaining only occupied cells. It converts a point cloud into a sparse voxel grid representation, storing only voxels that contain points. This method serves as the core voxelization step in pipelines converting point clouds to sparse voxel representations for downstream rendering or USD export.

### [Point Sampling from Mesh](3786-point-sampling-from-mesh.md)
Generates a discrete set of 3D coordinates on the surface of a polygon mesh. It bridges continuous mesh representations and discrete point-based volume data structures for downstream voxelization or neural field training. It differs from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) by generating points from a mesh source rather than consuming an existing point cloud, and from [Interior Voxelize](995-interior-voxelize.md) by targeting the surface boundary rather than the volume interior.

### [Ray Model Voxelization](4688-ray-model-voxelization.md)
Converts sparse LiDAR point clouds into dense 3D occupancy grids by tracing rays from sensor origins to detected points, classifying voxels as free, occupied, or unobserved. It differs from [Hard Voxelization](5345-hard-voxelization.md) by explicitly modeling the empty space between the sensor and the target points, rather than only marking voxels containing points, and from [Point Cloud to Gaussian Converter](1788-point-cloud-to-gaussian-converter.md) by producing discrete grid states rather than continuous density fields.

### [Hard Voxelization](5345-hard-voxelization.md)
Converts a continuous 3D point cloud into a discrete voxel grid by assigning each point to a specific voxel cell based on its spatial coordinates and a fixed voxel size [G]. This method serves as the foundational step for producing volume data from raw point clouds, building on [Point Cloud Voxelization](1000-point-cloud-voxelization.md) [G].

### [Voxel-based Void Ratio Computation](6679-voxel-based-void-ratio-computation.md)
Computes a void ratio field from a legacy ASCII VTK particle point cloud by binning particle centers into cubic voxels, calculating the solid volume via sphere-box intersection integration, and assigning the resulting void ratio to all particles in the voxel. It differs from [Hard Voxelization](5345-hard-voxelization.md) by accounting for partial sphere volumes at voxel boundaries using numerical integration, and from [Point Cloud to Gaussian Converter](1788-point-cloud-to-gaussian-converter.md) by focusing on precise geometric volume accounting rather than visual approximation.

### [Point Cloud Cropping and Downsampling](7057-point-cloud-cropping-and-downsampling.md)
Reduces the density of a point cloud by first restricting the points to a specific spatial volume (cropping) and then reducing the point count via either voxel-based binning or uniform random sampling. It serves as a preprocessing step for voxelization and registration, balancing computational load against geometric fidelity. It differs from [Octree Construction for Point Clouds](1070-octree-construction-for-point-clouds.md) by producing a flat list of points rather than a hierarchical structure, and from [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md) by using simpler spatial or uniform selection criteria rather than neighbor-aware masking.

### [Gaussian Voxelization](8884-gaussian-voxelization.md)
Refers to a family of methods that convert continuous 3D Gaussian primitives (splat) or point clouds into discrete voxel representations. These methods solve the problem of bridging unstructured, differentiable Gaussian splats with structured, grid-based neural networks or collision detection systems. They distinguish themselves from hard voxelization by using soft, probabilistic kernels or explicit sampling shapes to determine voxel occupancy and feature aggregation, rather than simple point-in-box tests.

### Two-stage training with adaptive point cloud generation
Splits training into two stages. After stage 1, generates additional point clouds using noise and UDF gradient descent, filters by UDF threshold, and rebuilds a KD-tree for denser sampling in stage 2 [S6, unofficial]. In `gen_extra_pointcloud`, sample points and samples. Compute offsets and standard deviation. Generate noise with std scaled by `low_range`. Add noise to points. Compute UDF and gradient. Project samples to surface. Filter by `udf_sample < df_filter`. Repeat until `num_points` reached. Concatenate and remove outliers [S6, unofficial]. After stage 1, concatenate ground truth points with generated points, build cKDTree, and update dataset for stage 2 training [S6, unofficial]. Parameters include `low_range` (scaling factor for noise standard deviation) and `outlier` (distance truncation threshold for removing outliers) [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for computing coordinates and $O(V \cdot M)$ for sparse storage updates, where $N$ is the number of points, $V$ is the number of occupied voxels, and $M$ is the max points per voxel [S3, unofficial]. Dense aggregation is $O(N)$ using segment operations [S5, unofficial].
*   **Memory Complexity:** Dense: $O(G_x G_y G_z)$ for the grid [S5, unofficial]. Sparse: $O(V \cdot M)$ for the voxel arrays, plus $O(G_x G_y G_z)$ for the coordinate-to-index map if used [S3, unofficial].
*   **Performance:** JIT-compiled versions (e.g., using Numba) significantly reduce Python overhead compared to pure Python loops [S3, unofficial].
*   **Strengths:** Fast, deterministic mapping; supports feature aggregation; handles sparse data efficiently [S3, S5, unofficial].
*   **Limitations:** Fixed resolution limits detail; boundary handling (discard vs. clamp) affects data integrity; max limits can truncate data [S3, S5, unofficial].
*   **When to use:** When a structured grid representation is required for neural networks (e.g., 3D CNNs) or efficient spatial queries [S3, S5, unofficial].
*   **When not to use:** When high-frequency details are lost due to voxel size, or when the point cloud is extremely sparse and a sparse structure is not supported [S3, unofficial].

## Background
Point cloud voxelization is a fundamental technique in 3D computer vision and graphics, originating from the need to process unstructured point cloud data (e.g., from LiDAR or depth cameras) using grid-based algorithms [S2][S4, unofficial]. It is widely used in autonomous driving for occupancy grid mapping [S3, unofficial], 3D object detection [S5, unofficial], and neural surface reconstruction [S6, unofficial]. The method evolved from simple occupancy grids to include feature aggregation and sparse storage optimizations to handle large-scale scenes [S3, S5, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S3] danielhavir/voxelize3d, https://github.com/danielhavir/voxelize3d, fetched 2026-10-27 (unofficial)
[S4] sgffoen/PointCloudVoxelization, https://github.com/sgffoen/PointCloudVoxelization, fetched 2026-10-27 (unofficial)
[S5] https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py#L305, https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py, fetched 2026-10-27 (unofficial)
[S6] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
