# Pointcloud to Sparse Voxel Grid Conversion

Also called: Open3D Voxel Grid Creation, Point Cloud Voxelization with pyntcloud, Pyntcloud Voxel Grid Creation, add_structure voxelgrid

## Overview
Pointcloud to Sparse Voxel Grid Conversion maps a set of 3D points into a sparse grid of voxel centers and associated features, retaining only occupied cells [S1, unofficial]. It converts a point cloud into a sparse voxel grid representation, storing only voxels that contain points [S2, unofficial]. This method serves as the core voxelization step in pipelines converting point clouds to sparse voxel representations for downstream rendering or USD export [S1, S2, S3, unofficial].

## How it works
**Inputs:**
- `points`: A tensor/array of shape $[N, 3]$ containing 3D coordinates (x, y, z) [S2, unofficial].
- `features` (optional): A tensor/array of shape $[N, F]$ containing per-point attributes (e.g., color, normals) [S2, unofficial].
- `voxel_size` or `grid_cell_size`: A scalar or vector $[s_x, s_y, s_z]$ defining the dimensions of each voxel cube [S1, S2, unofficial].
- `segment_func` (optional): An aggregation function (e.g., mean, max) for combining features within a voxel [S2, unofficial].

**Outputs:**
- `voxel_centers`: Coordinates of occupied voxel centers [S1, unofficial].
- `voxel_features`: Aggregated features for each occupied voxel, shape $[N', F]$ where $N'$ is the number of occupied voxels [S2, unofficial].
- `voxel_indices`: Integer indices $[N', 3]$ identifying the grid position of each occupied voxel [S2, unofficial].
- `segment_ids`: Mapping from original points to voxel segments [S2, unofficial].

**Data Structures:**
- **Voxel Grid**: A sparse representation of occupied cells. In Open3D, this is a `VoxelGrid` object storing `origin`, `voxel_size`, and a list of `Voxel` objects, each with `grid_index` (x, y, z) [S1, S3, unofficial]. In TensorFlow, this is represented by sparse tensors or hash maps mapping flattened indices to features [S2, unofficial].
- **Output Formats**:
    - CSV: Header `["x", "y", "z", "voxel_size"]` followed by rows of center coordinates and the voxel size used [S1, unofficial].
    - NPY: A dictionary with keys `'centers'` (numpy array of floats), `'voxel_size'` (float), and `'count'` (int) [S1, unofficial].

**Steps:**

1. **Load Point Cloud**: Load the input point cloud from files (e.g., `.ply`, `.pcd`, `.xyz`, `.csv`) into a numerical array format [S1, S3, unofficial]. For CSV/TXT, extract the first three numeric columns as x, y, z [S1, unofficial]. If the file lacks x,y,z columns and has fewer than 3 numeric columns, raise a `ValueError` [S1, unofficial]. If the point cloud is empty or cannot be loaded, raise a `ValueError` [S1, unofficial].

2. **Preprocessing (Optional)**:
    - **Downsampling**: If `downsample_first` is true and `downsample_size` > 0, apply voxel-based downsampling to the point cloud using `voxel_down_sample(downsample_size)` [S1, unofficial].
    - **Outlier Removal**: If `remove_outliers` is true, remove statistical outliers using `remove_statistical_outlier(nb_neighbors, std_ratio)` [S1, unofficial]. Typical defaults: `nb_neighbors=20`, `std_ratio=2.0` [S1, unofficial].

3. **Compute Voxel Indices**: Determine the grid index for each point. Let $P_i$ be the coordinates of point $i$, and $V_{size}$ be the voxel size [S2, unofficial]. Calculate the start location (minimum point coordinates):
    $$ L_{start} = \min_{i} (P_i) $$
    Compute the integer grid index for each point:
    $$ I_i = \lfloor \frac{P_i - L_{start}}{V_{size}} \rfloor $$
    where $\lfloor \cdot \rfloor$ denotes the floor function applied element-wise [S2, unofficial]. Convert 3D indices $I_i$ to a single flattened index if required by the aggregation function [S2, unofficial].

4. **Aggregate Features (Optional)**: If features are provided, group points by their voxel indices and aggregate using `segment_func` [S2, unofficial].
    $$ F_{voxel} = \text{segment\_func}(F_{points}, \text{segment\_ids}) $$
    Common functions include `tf.math.unsorted_segment_mean` [S2, unofficial]. Compute the unique voxel indices by taking the max over segments to handle any indexing collisions or to identify unique occupied voxels:
    $$ I_{unique} = \text{unsorted\_segment\_max}(I_{points}, \text{segment\_ids}, \text{num\_segments}) $$
    [S2, unofficial].

5. **Construct Sparse Grid**:
    - **Open3D Approach**: Create a `VoxelGrid` object using `VoxelGrid.create_from_point_cloud(point_cloud, voxel_size)` [S1, S3, unofficial]. This internally handles the mapping and storage of occupied voxels [S3, unofficial].
    - **Pyntcloud Approach**: Call `.add_structure("voxelgrid", grid_size_x, grid_size_y, grid_size_z)` on the point cloud object [S3, unofficial]. This generates voxel grid IDs for intersecting points [S3, unofficial]. Access the occupied voxel representation via `point_cloud.structures[voxelgrid_ids]` [S3, unofficial].
    - **TensorFlow Approach**: Return the aggregated `voxel_features`, `voxel_indices`, `segment_ids`, and `voxel_start_location` [S2, unofficial].

6. **Extract Voxel Centers**: For each occupied voxel with grid index $G = (g_x, g_y, g_z)$, calculate the center coordinates $C$:
    $$ C = L_{origin} + (G + 0.5) \times V_{size} $$
    where $L_{origin}$ is the origin of the voxel grid (often derived from $L_{start}$ or explicitly defined) [S1, unofficial]. In Open3D, this is done by iterating over `voxel_grid.get_voxels()` [S1, unofficial].

**Pseudocode (TensorFlow-style implementation):**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import tensorflow as tf

def pointcloud_to_sparse_voxel_grid(points, features, grid_cell_size, segment_func=tf.math.unsorted_segment_mean):
    """
    Converts a pointcloud into a sparse voxel grid.
    
    Args:
        points: tf.float32 tensor of size [N, 3].
        features: tf.float32 tensor of size [N, F].
        grid_cell_size: tf.float32 tensor of size [3] or scalar.
        segment_func: Aggregation function (e.g., tf.math.unsorted_segment_mean).
        
    Returns:
        voxel_features: tf.float32 tensor of size [N', F].
        voxel_indices: tf.int32 tensor of size [N', 3].
        segment_ids: tf.int32 tensor of size [N].
        voxel_start_location: tf.float32 tensor of size [3].
    """
    grid_cell_size = tf.convert_to_tensor(grid_cell_size, dtype=tf.float32)
    
    # Step 1: Compute start location (min of points)
    voxel_start_location = tf.reduce_min(points, axis=0)
    
    # Step 2: Compute integer voxel indices for each point
    voxel_xyz_indices = tf.cast(
        tf.math.floordiv(points - voxel_start_location, grid_cell_size),
        dtype=tf.int32
    )
    
    # Step 3: Compute flattened indices for pooling
    # [G] Flatten 3D indices to 1D segment IDs for pooling. 
    # In a production system, this uses a hash map or computes grid bounds.
    voxel_xyz_indices_flat = voxel_xyz_indices[:, 0] * 1000000 + voxel_xyz_indices[:, 1] * 1000 + voxel_xyz_indices[:, 2]
    _, segment_ids = tf.unique(voxel_xyz_indices_flat)
    num_segments = tf.reduce_max(segment_ids) + 1
    
    # Step 4: Aggregate features
    if features is not None:
        voxel_features = segment_func(features, segment_ids, num_segments)
    else:
        voxel_features = tf.zeros([0, tf.shape(features)[1]], dtype=features.dtype)
        
    # Step 5: Get unique voxel indices
    voxel_xyz_indices = tf.math.unsorted_segment_max(
        data=voxel_xyz_indices,
        segment_ids=segment_ids,
        num_segments=num_segments
    )
    
    return voxel_features, voxel_xyz_indices, segment_ids, voxel_start_location
```

**Parameters:**
- `voxel_size`: Size of each voxel cube. Default 0.05 [S1, unofficial].
- `grid_cell_size`: Size of grid cells in x, y, z dimensions [S2, unofficial].
- `downsample_size`: Size for initial voxel downsampling. Default 0.03 [S1, unofficial].
- `nb_neighbors`: Number of neighbors for statistical outlier removal. Default 20 [S1, unofficial].
- `std_ratio`: Standard deviation ratio for statistical outlier removal. Default 2.0 [S1, unofficial].
- `grid_size_x, grid_size_y, grid_size_z`: Dimensions specifying the resolution of the voxel grid along each axis (used in Pyntcloud) [S3, unofficial].

**Edge Cases and Pitfalls:**
- If the point cloud is empty or cannot be loaded, raise `ValueError` [S1, unofficial].
- If CSV/TXT files lack x,y,z columns and have fewer than 3 numeric columns, raise `ValueError` [S1, unofficial].
- If no valid points are found after parsing, raise `ValueError` [S1, unofficial].
- Precision issues may arise when computing floor divisions for large coordinate values; using float32 is standard but float64 may be needed for high-precision applications [G].

## Subtopics

### [DCX Points-to-Voxels](3783-dcx-points-to-voxels.md)
DCX Points-to-Voxels converts a set of 3D points (from PLY files or 3D Gaussian Splatting) into a sparse voxel grid by averaging point positions and colors within grid cells defined by a bounding box and resolution. It differs from simple occupancy voxelization by storing averaged geometric and color data per voxel, and from dense voxelization by using hash maps or sparse arrays to store only occupied cells. This method serves as the core voxelization step in pipelines converting point clouds to sparse voxel representations for downstream rendering or USD export.

## Operating characteristics
- **Time Complexity**: $O(N)$ for computing indices and aggregating features, where $N$ is the number of points [S2, unofficial].
- **Memory Complexity**: $O(N')$ where $N'$ is the number of occupied voxels, which is $\le N$ [S2, unofficial].
- **Measured Performance**: Not given in the sources.
- **Scaling**: Scales linearly with the number of points. Sparse storage ensures memory efficiency for sparse point clouds [S2, unofficial].
- **Strengths**: Efficient storage for sparse data; supports feature aggregation; compatible with various input formats [S1, S3, unofficial].
- **Limitations**: Not given in the sources.
- **When to use**: When converting point clouds to sparse voxel representations for rendering, analysis, or further processing [S1, S2, S3, unofficial].
- **When not to use**: When dense voxel representation is required, or when the point cloud is extremely dense and memory is constrained [G].

## Background
The method originates from various open-source libraries and research projects [S1, S2, S3, unofficial]. Open3D provides a `VoxelGrid` class for creating voxel grids from point clouds [S1, S3, unofficial]. Pyntcloud offers an `add_structure` method for voxelization [S3, unofficial]. TensorFlow-based implementations, such as those in Google Research's `tf3d` library, provide functions like `pointcloud_to_sparse_voxel_grid_unbatched` for efficient GPU-accelerated voxelization [S2, unofficial]. These tools are widely used in robotics, 3D modeling, and computer vision for tasks like occupancy grid generation and point cloud preprocessing [S1, unofficial].

## Sources
[S1] Guda108/3D-Voxelization-GUI, https://github.com/Guda108/3D-Voxelization-GUI, fetched 2026-10-27 (unofficial)
[S2] https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py#L305, https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py, fetched 2026-10-27 (unofficial)
[S3] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
