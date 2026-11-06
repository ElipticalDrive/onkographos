# Point Cloud Voxelization

Also called: Pointcloud to Dense Voxel Grid Conversion, Voxelization from Point Cloud

## Overview
Point Cloud Voxelization converts a discrete set of 3D points into a structured voxel grid, mapping continuous world-space coordinates to discrete integer grid indices and aggregating point attributes into grid cells [S2][S5, unofficial]. It solves the problem of transforming unstructured point data into a regular volume representation suitable for 3D convolutional neural networks or ray-marching renderers [S3, S5, unofficial]. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md), which projects geometry through screen-space primitives, this method operates directly in world space by binning points into spatial cells [S1, S5, unofficial]. It differs from [Hard Voxelization](5345-hard-voxelization.md) by explicitly handling feature aggregation and sparse data structures rather than simple binary occupancy [S3, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A point cloud array `points` of shape $(N, D)$, where $N$ is the number of points and $D \ge 3$ (first 3 columns are $x, y, z$ coordinates) [S3, S5, unofficial]. Optional features `features` of shape $(N, F)$ may accompany the points [S5, unofficial].
*   **Parameters:**
    *   `voxel_size`: Physical dimensions of a voxel, shape $(3,)$ or scalar [S3, S5, unofficial].
    *   `grid_range` or `start_location`/`end_location`: Bounding box defining the voxel grid extent [S3, S5, unofficial].
    *   `max_num_voxels`: Maximum number of voxels to allocate [S3, unofficial].
    *   `max_points_in_voxel`: Maximum points stored per voxel [S3, unofficial].
    *   `segment_func`: Aggregation function (e.g., mean, sum) for features [S5, unofficial].
*   **Output:** A voxel grid structure containing occupied voxel coordinates, associated point data/features, and occupancy counts [S3, S5, unofficial].

**Data Structures:**
1.  **Dense Sparse Map (Coordinate-to-Index):** A 3D array `coor_to_voxelidx` of shape $(Z, Y, X)$ initialized to $-1$, mapping voxel coordinates to a sparse linear index [S3, unofficial].
2.  **Voxel Buffer:** A 3D array `voxels` of shape $(M, K, D)$, where $M$ is `max_num_voxels`, $K$ is `max_points_in_voxel`, storing point data for active voxels [S3, unofficial].
3.  **Coordinate Buffer:** A 2D array `coordinates` of shape $(M, 3)$ storing the integer grid indices of active voxels [S3, unofficial].
4.  **Count Buffer:** A 1D array `num_points_per_voxel` of shape $(M,)$ storing the number of points currently stored in each voxel [S3, unofficial].

**Algorithm Steps:**
1.  **Grid Size Calculation:** Compute the number of voxels per axis:
    $$ grid\_size = \lfloor \frac{grid\_range[3:6] - grid\_range[0:3]}{voxel\_size} \rfloor $$
    Cast to integer [S3, unofficial]. Alternatively, if using `start_location` and `end_location`, use ceiling division [S5, unofficial].
2.  **Coordinate Discretization:** For each point $p_i$ with coordinates $(x, y, z)$, compute the initial voxel index:
    $$ idx = \lfloor \frac{p_i - origin}{voxel\_size} \rfloor $$
    Cast to integer [S3, S5, unofficial].
3.  **Boundary Handling:**
    *   *Discard Method:* Create a mask for points where $0 \le idx < grid\_size$ in all dimensions. Discard points outside this range [S3, unofficial].
    *   *Clamp Method:* Clamp indices to $[0, grid\_size - 1]$ for points outside bounds [S5, unofficial].
4.  **Sparse Indexing:** Iterate through valid points. For each point's voxel coordinate $c$:
    *   Look up `voxel_id = coor_to_voxelidx[c]`.
    *   If `voxel_id == -1` (new voxel):
        *   Assign `voxel_id = current_voxel_count`.
        *   Increment `current_voxel_count`.
        *   If `current_voxel_count > max_num_voxels`, stop processing [S3, unofficial].
        *   Store `c` in `coordinates[voxel_id]`.
        *   Update `coor_to_voxelidx[c] = voxel_id`.
5.  **Feature Aggregation/Storage:**
    *   Get `point_idx = num_points_per_voxel[voxel_id]`.
    *   If `point_idx < max_points_in_voxel`:
        *   Store point data in `voxels[voxel_id, point_idx]`.
        *   Increment `num_points_per_voxel[voxel_id]`.
    *   If `point_idx >= max_points_in_voxel`, discard the point [S3, unofficial].
    *   *Alternative Aggregation:* If using a dense grid with feature aggregation (e.g., TensorFlow), compute a flattened `segment_id` for each point:
        $$ segment\_id = idx_x \cdot (Y \cdot Z) + idx_y \cdot Z + idx_z $$
        Apply `segment_func` (e.g., `tf.math.unsorted_segment_mean`) to aggregate features into a dense grid of shape $(X, Y, Z, F)$ [S5, unofficial].

**Pseudocode (Sparse Implementation):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def point_cloud_voxelization(points, voxel_size, grid_range, max_points_in_voxel=60, max_num_voxels=20000):
    """
    Convert point cloud to sparse voxel grid.
    points: (N, D) array, first 3 cols are x,y,z
    voxel_size: (3,) array
    grid_range: (6,) array [min_x, min_y, min_z, max_x, max_y, max_z]
    """
    # 1. Compute grid size
    grid_size = np.floor((grid_range[3:] - grid_range[:3]) / voxel_size).astype(np.int32)
    
    # 2. Initialize data structures
    # coor_to_voxelidx shape: (Z, Y, X)
    coor_to_voxelidx = np.full((grid_size[2], grid_size[1], grid_size[0]), -1, dtype=np.int32)
    voxels = np.zeros((max_num_voxels, max_points_in_voxel, points.shape[1]), dtype=points.dtype)
    coordinates = np.zeros((max_num_voxels, 3), dtype=np.int32)
    num_points_per_voxel = np.zeros(max_num_voxels, dtype=np.int32)
    
    # 3. Discretize coordinates
    # points_copy[:, :3] are x,y,z
    coor = np.floor((points[:, :3] - grid_range[:3]) / voxel_size).astype(np.int32)
    
    # 4. Mask points within bounds
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
        # Note: Array indexing order may vary (X,Y,Z vs Z,Y,X). 
        # Here we assume coor is (x,y,z) and array is indexed [z,y,x] or similar.
        # Adjust tuple order based on specific implementation's axis convention.
        # Standard numpy (C-order) for (Z,Y,X) shape:
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
Constructs a hierarchical spatial data structure (octree) to organize point cloud objects for rendering, recursively subdividing space into eight octants until a leaf condition is met. It sets itself apart from flat voxel grids by enabling efficient CPU-based frustum culling and view-dependent sampling through aggregated node data, reducing the rendering load for large point clouds.

### [Point Cloud to Gaussian Converter](1788-point-cloud-to-gaussian-converter.md)
Transforms a discrete set of 3D points into a volume representation composed of isotropic Gaussian ellipsoids, enabling differentiable volume rendering of point-based data. It solves the problem of converting unstructured point clouds into a continuous density field suitable for ray marching, distinguishing itself from hard voxelization by using soft, probabilistic kernels defined by local point proximity. Unlike the Mesh to Isotropic Gaussian Converter, which relies on mesh topology (edges), this method determines kernel scale based on the spatial distribution of neighboring points.

### [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md)
Resamples a point cloud based on active and neighbor status to support partial, percentage, and full resampling strategies. It controls point density by selectively activating points tied to tetrahedral connectivity.

### [Pointcloud to Sparse Voxel Grid Conversion](3416-pointcloud-to-sparse-voxel-grid-conversion.md)
Maps a set of 3D points into a sparse grid of voxel centers and associated features, retaining only occupied cells. It differs from dense voxelization by using hash maps or sparse arrays to store only occupied cells, and from simple occupancy voxelization by optionally storing averaged geometric and color data per voxel. This method serves as the core voxelization step in pipelines converting point clouds to sparse voxel representations for downstream rendering or USD export.

### [Point Sampling from Mesh](3786-point-sampling-from-mesh.md)
Generates a discrete set of 3D coordinates uniformly distributed on the surface of a polygon mesh. It serves as a bridge between continuous mesh representations and discrete point-based volume data structures, often acting as an intermediate step for voxelization or neural implicit field training. Unlike rasterization-based methods that project geometry to screen space, this method operates directly in world space by sampling the mesh surface geometry. It is distinct from Point Cloud Voxelization because it generates the points from a mesh source rather than consuming an existing point cloud, and from Interior Voxelize because it targets the surface boundary rather than the volume interior.

### [Ray Model Voxelization](4688-ray-model-voxelization.md)
Converts sparse LiDAR point clouds into dense 3D occupancy grids by tracing rays from sensor origins to detected points, classifying voxels as free, occupied, or unobserved. It differs from Hard Voxelization by explicitly modeling the empty space between the sensor and the target points, rather than only marking voxels containing points, and from Point Cloud to Gaussian Converter by producing discrete grid states rather than continuous density fields.

### [Hard Voxelization](5345-hard-voxelization.md)
Converts a continuous 3D point cloud into a discrete voxel grid by assigning each point to a specific voxel cell based on its spatial coordinates and a fixed voxel size [G]. This method serves as the foundational step for producing volume data from raw point clouds, building on Point Cloud Voxelization.

### [Voxel-based Void Ratio Computation](6679-voxel-based-void-ratio-computation.md)
Computes a void ratio field from a legacy ASCII VTK particle point cloud by binning particle centers into cubic voxels, calculating the solid volume via sphere-box intersection integration, and assigning the resulting void ratio to all particles in the voxel. It differs from Hard Voxelization by accounting for partial sphere volumes at voxel boundaries using numerical integration, and from Point Cloud to Gaussian Converter by focusing on precise geometric volume accounting rather than visual approximation.

### [Point Cloud Cropping and Downsampling](7057-point-cloud-cropping-and-downsampling.md)
Prepares point clouds for voxelization or registration by removing points outside a specified volume and reducing point density to a manageable size. It solves the problem of excessive memory usage and computational cost in downstream tasks like ICP registration or sparse voxel grid construction by ensuring the input point cloud fits within hardware limits and spatial resolution constraints. It is distinguished from sibling methods like Hard Voxelization by operating on the raw point set prior to voxel assignment, rather than during the voxelization process itself.

### Nearest Neighbor Splatting
A splatting method where a value at a continuous position is written to the single nearest voxel by rounding coordinates to integers [S6, unofficial]. Implementation involves mapping continuous coordinates $(x,y,z)$ to discrete indices $(i,j,k)$ via rounding and adding the value to the buffer if within bounds [S6, unofficial]. This method suffers from obvious aliasing problems [S6, unofficial].

### Voxels to Points Projection
Projects features from a dense voxel grid back to specific points using pre-computed segment IDs [S5, unofficial]. It reshapes the voxel grid to 2D, then gathers features for each point using its segment ID, effectively reversing the aggregation step [S5, unofficial].

### Two-stage training with adaptive point cloud generation
Splits training into two stages. After stage 1, generates additional point clouds using noise and UDF gradient descent, filters by UDF threshold, and rebuilds a KD-tree for denser sampling in stage 2 [S7, unofficial]. It generates noise scaled by a `low_range` parameter, projects samples to the surface, and filters by a distance threshold `outlier` to remove outliers [S7, unofficial].

### 3D Gaussian Voxelization
Converts a set of 3D Gaussians into a set of unique voxels and their associated features for efficient processing by a 3D network [S8, unofficial]. It determines the number of voxels per Gaussian based on a `SAMPLE_SHAPE` (e.g., "volume", "center"), generates voxel coordinates and features, and deduplicates them to return unique coordinates, aggregated features, and inverse indices [S8, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for coordinate discretization and boundary checking, where $N$ is the number of points. The sparse indexing step is $O(N)$ on average if hash maps are used, or $O(N)$ with a fixed-size grid lookup table [S3, S5, unofficial].
*   **Memory Complexity:** $O(M \cdot K)$ for the sparse voxel buffer, where $M$ is the number of occupied voxels and $K$ is `max_points_in_voxel`. The dense coordinate map requires $O(X \cdot Y \cdot Z)$ memory [S3, unofficial].
*   **Performance:** JIT-compiled implementations (e.g., Numba) significantly reduce Python overhead, offering near-C performance [S3, unofficial].
*   **Strengths:** Efficiently handles large point clouds; supports feature aggregation; compatible with GPU acceleration [S3, S5, unofficial].
*   **Limitations:** Fixed voxel size limits resolution adaptability; sparse structures may have high memory overhead for very dense grids; boundary handling (discard vs. clamp) affects data integrity [S3, S5, unofficial].
*   **When to use:** When converting point clouds to regular grids for CNNs or sparse voxel renderers.
*   **When not to use:** When adaptive resolution is required (use Octree or View-Adaptive Voxelization) or when continuous density is needed (use Point Cloud to Gaussian Converter).

## Background
Point cloud voxelization is a fundamental technique in 3D computer vision and graphics, enabling the use of 2D/3D convolutional networks on point-based data. It originated from the need to process LiDAR and depth sensor data efficiently. Libraries like Open3D provide high-level APIs for this operation [S2]. JIT-compiled versions have been developed to optimize performance for large-scale datasets [S3, unofficial]. The method is widely used in autonomous driving, 3D reconstruction, and neural rendering pipelines [S1, S3, S5, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S3] danielhavir/voxelize3d, https://github.com/danielhavir/voxelize3d, fetched 2026-10-27 (unofficial)
[S4] sgffoen/PointCloudVoxelization, https://github.com/sgffoen/PointCloudVoxelization, fetched 2026-10-27 (unofficial)
[S5] https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py#L305, https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py, fetched 2026-10-27 (unofficial)
[S6] Production Volume Rendering, https://cg.informatik.uni-freiburg.de/intern/seminar/volumeRenderingProductionFundamentals2011.pdf, fetched 2026-10-27 (unofficial)
[S7] jjjkkyz/DCx, https://github.com/jjjkkyz/DCx, fetched 2026-10-26 (unofficial)
[S8] Quantile Rendering: Efficiently Embedding High-dimensional Feature on 3D Gaussian Splatting, https://doi.org/10.48550/arxiv.2512.20927, fetched 2026-10-26 (unofficial)
