# Point Cloud Voxelization

Also called: Pointcloud to Dense Voxel Grid Conversion, Voxelization from Point Cloud

## Overview
Point Cloud Voxelization converts an unstructured set of 3D points into a discrete volumetric grid representation suitable for rendering or processing [S2][S5, unofficial]. It solves the problem of mapping continuous, irregular point data into a regular structure by discretizing space and aggregating point attributes into grid cells [S5, unofficial]. Unlike [Voxel Grid from Mesh (Fill Method)](1013-voxelgrid-from-mesh-fill-method.md) which fills interior volumes, this method primarily marks cells containing points and aggregates their features, building on the shared principles of [Voxelization](12-voxelization.md) [G].

## How it works
**Inputs and Outputs:**
*   **Input:** A point cloud array `points` of shape $(N, D)$, where the first 3 columns are coordinates $(x, y, z)$ and the remaining $D-3$ columns are features (e.g., color, intensity) [S3, S5, unofficial].
*   **Parameters:**
    *   `voxel_size`: Array of shape $(3,)$ specifying the physical dimensions (width, length, height) of each voxel [S3, S5, unofficial].
    *   `grid_range` or `start_location`/`end_location`: Defines the spatial bounds of the voxel grid. `grid_range` is shape $(6,)$ as $[min_x, min_y, min_z, max_x, max_y, max_z]$ [S3, unofficial]. Alternatively, `start_location` (shape $(3,)$) and `end_location` (shape $(3,)$) define the min and max coordinates [S5, unofficial].
    *   `max_num_voxels`: Integer limit on the total number of voxels in the output (sparse representation) [S3, unofficial].
    *   `max_points_in_voxel`: Integer limit on points stored per voxel (sparse representation) [S3, unofficial].
    *   `segment_func`: Aggregation function for features (e.g., mean, sum, min, max) [S5, unofficial].
*   **Output:**
    *   **Dense:** A tensor of shape $(N_x, N_y, N_z, F)$ containing aggregated voxel features [S5, unofficial].
    *   **Sparse:** A tuple `(voxels, coordinates, num_points_per_voxel)` where `voxels` is shape $(V, M, D)$, `coordinates` is shape $(V, 3)$, and `num_points_per_voxel` is shape $(V,)$ [S3, unofficial].

**Data Structures:**
1.  **Dense Layout:** A 4D array where indices $(i, j, k)$ correspond to voxel grid coordinates.
2.  **Sparse Layout:**
    *   `coor_to_voxelidx`: A 3D lookup table of shape $(N_z, N_y, N_x)$ initialized to $-1$, mapping grid coordinates to a sparse index [S3, unofficial].
    *   `voxels`: A pre-allocated buffer of shape $(max\_num\_voxels, max\_points\_in\_voxel, D)$ [S3, unofficial].
    *   `coordinates`: A buffer of shape $(max\_num\_voxels, 3)$ storing the grid coordinates of active voxels [S3, unofficial].
    *   `num_points_per_voxel`: A buffer of shape $(max\_num\_voxels)$ storing the count of points in each active voxel [S3, unofficial].

**Steps:**
1.  **Grid Size Calculation:** Compute the dimensions of the voxel grid.
    *   Using `grid_range`: $N_x = \lfloor (max_x - min_x) / v_x \rfloor$, etc. [S3, unofficial].
    *   Using `start/end`: $N_x = \lceil (end_x - start_x) / v_x \rceil$ [S5, unofficial].
2.  **Coordinate Discretization:** Map each point $(x, y, z)$ to integer grid indices $(i, j, k)$.
    *   $i = \lfloor (x - min_x) / v_x \rfloor$ [S3, S5, unofficial].
    *   Note: Some implementations use 1-based indexing or add offsets depending on the language (e.g., Julia) [S4, unofficial].
3.  **Boundary Handling:**
    *   **Discard:** Points with indices outside $[0, N_x-1]$ are discarded [S3, unofficial].
    *   **Clamp:** Points outside bounds are clamped to the nearest edge voxel (index 0 or $N-1$) [S5, unofficial].
4.  **Feature Aggregation (Dense):**
    *   Compute a 1D segment ID for each point: $seg\_id = i \cdot (N_y N_z) + j \cdot N_z + k$ [S5, unofficial].
    *   Apply `segment_func` (e.g., `unsorted_segment_mean`) to aggregate features from `points` into the dense grid based on `seg_id` [S5, unofficial].
5.  **Sparse Mapping:**
    *   Initialize `coor_to_voxelidx` with $-1$ [S3, unofficial].
    *   Iterate through valid points. For point $p$ at grid coord $c$:
        *   Lookup `voxel_id = coor_to_voxelidx[c]`.
        *   If `voxel_id == -1`:
            *   Assign `voxel_id = current_voxel_count`.
            *   Increment `current_voxel_count`.
            *   If `current_voxel_count > max_num_voxels`, stop processing [S3, unofficial].
            *   Set `coor_to_voxelidx[c] = voxel_id`.
            *   Store $c$ in `coordinates[voxel_id]`.
        *   Store point $p$ in `voxels[voxel_id, num_points_per_voxel[voxel_id]]`.
        *   Increment `num_points_per_voxel[voxel_id]`.
        *   If `num_points_per_voxel[voxel_id] >= max_points_in_voxel`, discard subsequent points for this voxel [S3, unofficial].

**Pseudocode (Sparse, Numba-style):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def voxelize_sparse(points, voxel_size, grid_range, max_points_in_voxel=60, max_num_voxels=20000):
    """
    points: (N, D) array, first 3 cols are xyz
    voxel_size: (3,) array
    grid_range: (6,) array [min_x, min_y, min_z, max_x, max_y, max_z]
    """
    # 1. Compute grid size
    grid_size = np.floor((grid_range[3:] - grid_range[:3]) / voxel_size).astype(np.int32)
    
    # 2. Initialize structures
    # coor_to_voxelidx shape: (z, y, x) to match typical memory layout or (x,y,z) depending on implementation.
    # Source [S3, unofficial] uses (grid_size[2], grid_size[1], grid_size[0]) i.e., (z, y, x)
    coor_to_voxelidx = np.full((grid_size[2], grid_size[1], grid_size[0]), -1, dtype=np.int32)
    voxels = np.zeros((max_num_voxels, max_points_in_voxel, points.shape[1]), dtype=points.dtype)
    coordinates = np.zeros((max_num_voxels, 3), dtype=np.int32)
    num_points_per_voxel = np.zeros(max_num_voxels, dtype=np.int32)
    
    # 3. Discretize coordinates
    # points_copy[:, :3] are xyz
    coor = np.floor((points[:, :3] - grid_range[:3]) / voxel_size).astype(np.int32)
    
    # 4. Mask points within bounds
    # Check 0 <= coor < grid_size for all dims
    mask = (
        (coor[:, 0] >= 0) & (coor[:, 0] < grid_size[0]) &
        (coor[:, 1] >= 0) & (coor[:, 1] < grid_size[1]) &
        (coor[:, 2] >= 0) & (coor[:, 2] < grid_size[2])
    )
    coor = coor[mask]
    points_valid = points[mask]
    
    # 5. Iterate and fill
    voxel_num = 0
    for i, c in enumerate(coor):
        # Note: Source [S3, unofficial] reverses axis order for storage/access in some variants, 
        # but here we assume c is (x,y,z) and access coor_to_voxelidx[z,y,x]
        # Adjust indexing based on specific layout convention. 
        # Standardizing to c[0]=x, c[1]=y, c[2]=z:
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
            voxels[voxel_id, n_pts] = points_valid[i]
            num_points_per_voxel[voxel_id] += 1
            
    return voxels[:voxel_num], coordinates[:voxel_num], num_points_per_voxel[:voxel_num]
```

## Subtopics

### [Octree Construction for Point Clouds](1070-octree-construction-for-point-clouds.md)
Constructs a hierarchical spatial data structure (octree) to organize point clouds, recursively subdividing space into eight octants until a leaf condition is met. It differs from flat voxel grids by maintaining a list of points at each node and aggregating their average position and color for spatial queries. Choose this when memory efficiency for sparse data is critical or when multi-resolution queries are needed. Link: [Octree Construction for Point Clouds](1070-octree-construction-for-point-clouds.md).

### [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md)
Reduces point cloud density by selectively discarding points based on their topological relationship to active elements (e.g., tetrahedra). It preserves points critical to the structure (neighbors of active elements) while removing redundant ones, using explicit neighbor masks derived from the grid structure rather than random selection. Choose this for compression while maintaining geometric fidelity. Link: [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md).

### [Voxels to Points Projection](3172-voxels-to-points-projection.md)
Maps features stored in a dense voxel grid back to the original points in a point cloud using pre-computed segment IDs or point-to-voxel mappings. It solves the problem of recovering per-point features after aggregation, enabling downstream tasks like point-wise classification. Unlike Hard Voxelization which aggregates points into voxels, this reverses the flow. Choose this when feature propagation from grid to points is required. Link: [Voxels to Points Projection](3172-voxels-to-points-projection.md).

### [Pointcloud to Sparse Voxel Grid Conversion](3416-pointcloud-to-sparse-voxel-grid-conversion.md)
Maps a set of 3D points into a sparse grid of voxel centers and associated features, retaining only occupied cells. It converts a point cloud into a sparse voxel grid representation, storing only voxels that contain points. This serves as the core voxelization step in pipelines converting point clouds to sparse voxel representations for downstream rendering or USD export. Link: [Pointcloud to Sparse Voxel Grid Conversion](3416-pointcloud-to-sparse-voxel-grid-conversion.md).

### [Point Sampling from Mesh](3786-point-sampling-from-mesh.md)
Generates a discrete set of 3D coordinates on the surface of a polygon mesh. It bridges continuous mesh representations and discrete point-based volume data structures for downstream voxelization or neural field training. It differs from Point Cloud Voxelization by generating points from a mesh source rather than consuming an existing point cloud. Link: [Point Sampling from Mesh](3786-point-sampling-from-mesh.md).

### [Ray Model Voxelization](4688-ray-model-voxelization.md)
Converts sparse LiDAR point clouds into dense 3D occupancy grids by tracing rays from sensor origins to detected points, classifying voxels as free, occupied, or unobserved. It differs from Hard Voxelization by explicitly modeling the empty space between the sensor and the target points. Choose this for LiDAR-based scene reconstruction. Link: [Ray Model Voxelization](4688-ray-model-voxelization.md).

### [Hard Voxelization](5345-hard-voxelization.md)
Converts a continuous 3D point cloud into a discrete voxel grid by assigning each point to a specific voxel cell based on its spatial coordinates and a fixed voxel size [G]. This method serves as the foundational step for producing volume data from raw point clouds, building on Point Cloud Voxelization [G]. Link: [Hard Voxelization](5345-hard-voxelization.md).

### [Voxel-based Void Ratio Computation](6679-voxel-based-void-ratio-computation.md)
Computes a void ratio field from a legacy ASCII VTK particle point cloud by binning particle centers into cubic voxels, calculating the solid volume via sphere-box intersection integration, and assigning the resulting void ratio to all particles in the voxel. It differs from Hard Voxelization by accounting for partial sphere volumes at voxel boundaries using numerical integration. Link: [Voxel-based Void Ratio Computation](6679-voxel-based-void-ratio-computation.md).

### [Point Cloud Cropping and Downsampling](7057-point-cloud-cropping-and-downsampling.md)
Crops a point cloud using a specified volume and downsamples it using either voxel or uniform downsampling. It differs from its siblings by providing a unified interface for spatial filtering and density reduction, which serves as a critical preprocessing step for downstream tasks such as ICP registration and voxelization. Link: [Point Cloud Cropping and Downsampling](7057-point-cloud-cropping-and-downsampling.md).

### [Gaussian Voxelization](8884-gaussian-voxelization.md)
Refers to a family of methods that convert continuous 3D Gaussian primitives (splat) or point clouds into discrete voxel representations. These methods solve the problem of bridging unstructured, differentiable Gaussian splats with structured, grid-based neural networks or collision detection systems. They distinguish themselves from hard voxelization by using soft, probabilistic kernels or explicit sampling shapes to determine voxel occupancy and feature aggregation, rather than simple point-in-box tests. Link: [Gaussian Voxelization](8884-gaussian-voxelization.md).

### Two-stage training with adaptive point cloud generation
Splits training into two stages. After stage 1, generates additional point clouds using noise and UDF gradient descent, filters by UDF threshold, and rebuilds a KD-tree for denser sampling in stage 2 [S6, unofficial]. In `gen_extra_pointcloud`, sample points and samples, compute offsets and standard deviation, generate noise with std scaled by `low_range`, add noise to points, compute UDF and gradient, project samples to surface, filter by `udf_sample < df_filter`, and repeat until `num_points` reached [S6, unofficial]. After stage 1, concatenate ground truth points with generated points, build cKDTree, and update dataset for stage 2 training [S6, unofficial]. Parameters include `low_range` (scaling factor for noise std) and `outlier` (distance truncation threshold for removing outliers) [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for coordinate discretization and masking. Sparse mapping is $O(N)$ amortized if hash lookups are $O(1)$, but the explicit array lookup in `coor_to_voxelidx` is $O(1)$ per point. Aggregation is $O(N)$ for dense segment operations.
*   **Memory Complexity:**
    *   Dense: $O(N_x N_y N_z)$ for the grid, regardless of point count.
    *   Sparse: $O(V \cdot M)$ where $V$ is the number of occupied voxels and $M$ is `max_points_in_voxel`. The lookup table `coor_to_voxelidx` requires $O(N_x N_y N_z)$ space [S3, unofficial].
*   **Performance:** JIT-compiled versions (e.g., Numba) significantly reduce Python overhead [S3, unofficial].
*   **Strengths:** Simple, fast, and parallelizable. Produces regular data structures suitable for CNNs and volume rendering.
*   **Limitations:** Memory usage grows cubically with resolution for dense grids. Sparse grids require large lookup tables for the coordinate-to-index mapping. Points outside bounds are either discarded or clamped, potentially losing information or introducing bias [S3, S5, unofficial].
*   **When to use:** When the scene fits within a known bounding box and a regular grid is required for downstream processing (e.g., 3D CNNs).
*   **When not to use:** For highly sparse, large-scale outdoor scenes where sparse structures (like Octrees or Hash Tables) are more memory-efficient, or when continuous representations are preferred.

## Background
Point cloud voxelization is a fundamental technique in 3D computer vision and graphics, enabling the conversion of unstructured sensor data (LiDAR, photogrammetry) into structured formats for processing. It is widely used in autonomous driving (e.g., PointPillars, VoxelNet), 3D reconstruction, and neural rendering. The method described here, particularly the sparse variant with explicit coordinate mapping, is common in deep learning pipelines for 3D object detection and segmentation [S3, S5, unofficial]. The dense variant with feature aggregation is standard in libraries like Open3D [S2] and TensorFlow-based research [S5, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S3] danielhavir/voxelize3d, https://github.com/danielhavir/voxelize3d, fetched 2026-10-27 (unofficial)
[S4] sgffoen/PointCloudVoxelization, https://github.com/sgffoen/PointCloudVoxelization, fetched 2026-10-27 (unofficial)
[S5] https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py#L305, https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py, fetched 2026-10-27 (unofficial)
[S6] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
