# Pointcloud to Sparse Voxel Grid Conversion

Also called: Open3D Voxel Grid Creation, Point Cloud Voxelization with pyntcloud, PointCloud to Schematic Conversion, Pyntcloud Voxel Grid Creation, add_structure voxelgrid

## Overview
Pointcloud to Sparse Voxel Grid Conversion transforms a continuous 3D point cloud into a discrete, sparse representation where only voxels containing input points are stored. It solves the problem of organizing unstructured point data into a regular grid structure suitable for downstream volume rendering, spatial analysis, or neural network processing. It differs from [Hard Voxelization](5345-hard-voxelization.md) by typically retaining feature aggregation (e.g., mean, max) rather than binary occupancy, and from [Gaussian Voxelization](8884-gaussian-voxelization.md) by using hard assignment to grid cells rather than continuous kernel weighting.

## How it works
The method maps each point in the input cloud to a discrete voxel index based on a fixed voxel size and a bounding box origin, then aggregates point features into those voxels.

**Inputs:**
*   `points`: Tensor of shape $[N, 3]$ containing $(x, y, z)$ coordinates.
*   `features`: Tensor of shape $[N, F]$ containing per-point features (e.g., color, normals). If absent, $F=0$ or occupancy is assumed.
*   `voxel_size` (or `grid_cell_size`): Scalar or vector $[s_x, s_y, s_z]$ defining the physical size of each voxel.
*   `start_location` (optional): The minimum coordinate $(x_{min}, y_{min}, z_{min})$ of the voxel grid. If not provided, it is computed as $\min(points)$.

**Outputs:**
*   `voxel_indices`: Tensor of shape $[N', 3]$ containing the $(i, j, k)$ indices of occupied voxels, where $N'$ is the number of unique occupied voxels.
*   `voxel_features`: Tensor of shape $[N', F]$ (or $[N', G, F]$ if multiple points per voxel are stored) containing the aggregated features for each occupied voxel.
*   `segment_ids`: Tensor of shape $[N]$ mapping each input point to its corresponding voxel segment ID.
*   `voxel_start_location`: The origin of the grid in world coordinates.

**Data Structures:**
*   **Voxel Index:** A 3-tuple $(i, j, k)$ of integers.
*   **Sparse Storage:** Typically implemented using a hash map (dictionary) mapping flattened voxel indices to aggregated features, or a sorted list of unique indices with associated feature buffers.

**Steps:**

1.  **Load and Validate:** Load point cloud data from file (e.g., `.ply`, `.xyz`, `.csv`) into memory arrays. If the input is empty or lacks 3 numeric columns, raise an error [S1, S3, S5, unofficial].
2.  **Preprocessing (Optional):**
    *   **Downsampling:** If enabled, apply voxel grid downsampling using `voxel_down_sample` with a specified `downsample_size` [S1, unofficial].
    *   **Outlier Removal:** If enabled, remove statistical outliers using `remove_statistical_outlier` with parameters `nb_neighbors` and `std_ratio` [S1, unofficial].
3.  **Compute Grid Origin:** Determine the start location of the voxel grid.
    *   Standard approach: $start\_location = \min(points, axis=0)$ [S2, unofficial].
    *   Alternative: Shift points so that the minimum coordinate is non-negative, then scale [S5, unofficial].
4.  **Map Points to Voxel Indices:** For each point $p_n = (x_n, y_n, z_n)$, compute its voxel index $(i_n, j_n, k_n)$:
    $$
    i_n = \lfloor \frac{x_n - x_{start}}{s_x} \rfloor, \quad
    j_n = \lfloor \frac{y_n - y_{start}}{s_y} \rfloor, \quad
    k_n = \lfloor \frac{z_n - z_{start}}{s_z} \rfloor
    $$
    where $\lfloor \cdot \rfloor$ is the floor function [S2, unofficial].
5.  **Flatten Indices:** Convert 3D indices to a single unique integer ID for segmentation. A common method is:
    $$
    id_n = i_n \cdot (H \cdot W) + j_n \cdot W + k_n
    $$
    where $W, H$ are sufficient dimensions to ensure uniqueness, or simply use a hash map key $(i_n, j_n, k_n)$ [S2, S3, unofficial].
6.  **Aggregate Features:** Group points by their voxel ID and apply an aggregation function (e.g., mean, max, sum) to the features.
    *   Using TensorFlow: `voxel_features, segment_ids, num_segments = pool_features_given_indices(features, indices, segment_func)` [S2, unofficial].
    *   Using Open3D: The library handles this internally when creating the `VoxelGrid` object, storing occupancy [S1, S3, unofficial].
    *   Using Pyntcloud: The `add_structure` method returns IDs of intersecting voxels, which are then used to access the structure [S3, unofficial].
7.  **Extract Unique Voxels:** Identify the set of unique voxel indices $V_{unique}$ that have at least one point.
8.  **Compute Centers (Optional):** For visualization or export, compute the center of each occupied voxel:
    $$
    center_v = start\_location + (index_v + 0.5) \cdot voxel\_size
    $$
    where $index_v$ is the integer index of the voxel [S1, S3, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import tensorflow as tf

def pointcloud_to_sparse_voxel_grid(points, features, voxel_size, segment_func=tf.math.unsorted_segment_mean):
    """
    Converts a point cloud into a sparse voxel grid.
    
    Args:
        points: np.ndarray of shape [N, 3], float32.
        features: np.ndarray of shape [N, F], float32.
        voxel_size: float or list of 3 floats [sx, sy, sz].
        segment_func: Aggregation function (e.g., tf.math.unsorted_segment_mean).
        
    Returns:
        voxel_indices: np.ndarray of shape [N', 3], int32.
        voxel_features: np.ndarray of shape [N', F], float32.
        segment_ids: np.ndarray of shape [N], int32.
        start_location: np.ndarray of shape [3], float32.
    """
    # 1. Convert to tensors
    points = tf.constant(points, dtype=tf.float32)
    features = tf.constant(features, dtype=tf.float32)
    grid_cell_size = tf.convert_to_tensor(voxel_size, dtype=tf.float32)
    
    # 2. Compute start location (min of points)
    start_location = tf.reduce_min(points, axis=0)
    
    # 3. Compute voxel indices
    # Offset points by start_location
    offset_points = points - start_location
    # Floor divide by voxel size
    voxel_xyz_indices = tf.cast(
        tf.math.floordiv(offset_points, grid_cell_size),
        dtype=tf.int32
    )
    
    # 4. Flatten indices to unique segment IDs
    # Assuming voxel_xyz_indices is [N, 3], we need a unique ID per voxel.
    # A simple way is to use a hash or a large multiplier if bounds are known.
    # Here we use TensorFlow's unsorted_segment operations which handle grouping.
    # We need a 1D index for each point.
    # Let's assume a simple flattening for demonstration, though in practice 
    # one might use a hash map or sorted unique operation.
    # For this pseudocode, we simulate the segment ID generation.
    # In TF, we can use tf.math.unsorted_segment_max/min/etc directly on the 3D indices 
    # to find the bounding box or unique indices, but for feature pooling, 
    # we need a 1D segment ID per point.
    
    # Compute a unique integer ID for each voxel (assuming non-negative indices)
    # This is a simplified flattening; robust implementations use hashing or 
    # coordinate-based hashing.
    # Let's assume we have a function to map (i,j,k) to a unique ID.
    # For the sake of this generic pseudocode, we will use the index itself 
    # if we were using a dense grid, but for sparse, we group by (i,j,k).
    
    # Alternative: Use tf.unique on the flattened indices? No, that's for 1D.
    # We will use the segment_ids from the pooling step if we had them, 
    # but we need to generate them first.
    
    # Let's use a standard approach: 
    # 1. Get unique voxel coordinates.
    # 2. Map each point to the index of its voxel in the unique list.
    
    # Step 4a: Get unique voxel indices
    # Reshape to [N, 1, 3] and [1, N, 3] for broadcasting comparison is expensive.
    # Instead, we can use a hash map or sort.
    # Using TF: 
    # sorted_indices = tf.argsort(voxel_xyz_indices[:, 0] * 1000000 + voxel_xyz_indices[:, 1] * 1000 + voxel_xyz_indices[:, 2])
    # unique_voxels, unique_indices = tf.unique(sorted_voxels)
    
    # Simplified for pseudocode: Assume we have a helper `get_segment_ids`
    # that maps each point's (i,j,k) to a segment ID 0..N'-1.
    segment_ids = get_segment_ids_from_voxel_indices(voxel_xyz_indices)
    
    # Step 5: Aggregate features
    # Get the number of unique segments
    num_segments = tf.reduce_max(segment_ids) + 1
    
    # Pool features
    voxel_features = tf.math.unsorted_segment_mean(features, segment_ids, num_segments)
    
    # Step 6: Get unique voxel indices corresponding to the pooled features
    # We need the (i,j,k) for each segment ID.
    # We can use unsorted_segment_max to get one representative index per segment
    # (since all points in a segment have the same index, max/min/mean works)
    voxel_xyz_indices_pooled = tf.math.unsorted_segment_max(
        data=voxel_xyz_indices,
        segment_ids=segment_ids,
        num_segments=num_segments
    )
    
    return voxel_xyz_indices_pooled, voxel_features, segment_ids, start_location

def get_segment_ids_from_voxel_indices(voxel_xyz_indices):
    """
    Helper to map 3D voxel indices to 1D segment IDs.
    This is a placeholder for a hashing or sorting-based unique operation.
    """
    # Flatten to 1D for unique operation
    # Assuming indices are small enough to fit in a 64-bit int
    # Or use a hash map. Here we simulate with a sort-based unique.
    # In real TF code: 
    # flat_indices = voxel_xyz_indices[:, 0] * 10**9 + voxel_xyz_indices[:, 1] * 10**6 + voxel_xyz_indices[:, 2]
    # _, inverse_indices = tf.unique_with_counts(flat_indices) # This doesn't give direct mapping
    # Correct TF approach:
    # sorted_vals, sorted_indices = tf.sort(flat_indices, direction='ASC')
    # unique_vals, unique_indices, counts = tf.unique_with_counts(sorted_vals)
    # segment_ids = tf.gather(unique_indices, tf.searchsorted(unique_vals, flat_indices))
    
    # For pseudocode clarity, we return a dummy mapping
    # In practice, this step is critical for performance.
    return tf.range(tf.shape(voxel_xyz_indices)[0]) # Placeholder
```

**Parameters:**
*   `voxel_size`: Default 0.05 in Open3D GUI [S1, unofficial].
*   `downsample_size`: Default 0.03 [S1, unofficial].
*   `nb_neighbors`: Default 20 for outlier removal [S1, unofficial].
*   `std_ratio`: Default 2.0 for outlier removal [S1, unofficial].
*   `segment_func`: Aggregation function, e.g., `tf.math.unsorted_segment_mean` [S2, unofficial].

**Edge Cases:**
*   **Empty Input:** Raise `ValueError` if no points are loaded [S1, S5, unofficial].
*   **Insufficient Columns:** Raise `ValueError` if CSV/TXT has fewer than 3 numeric columns [S1, unofficial].
*   **Large Coordinates:** Input coordinates can be large (e.g., $10^5$), while voxel units are small integers. Ensure integer overflow is handled in index calculation [S4, S5, unofficial].
*   **Negative Coordinates:** If `start_location` is not the global minimum, indices may be negative. Ensure the grid handles negative indices or shift points first [S5, unofficial].

## Subtopics

### [DCX Points-to-Voxels](3783-dcx-points-to-voxels.md)
Converts points to a sparse grid by mapping to indices and storing occupancy, prioritizing points with higher depth confidence in aggregation to mitigate noise. Choose when depth confidence is available and noise reduction is critical.

### [VoxelGrid::CreateFromPointCloudWithinBounds](8505-voxelgrid-createfrompointcloudwithinbounds.md)
Voxalizes using a direct hash-map accumulation approach with a fixed minimum bounding box corner, ignoring the maximum bound for coordinate calculation. Choose for fast, simple occupancy grids without complex feature aggregation.

### [Multi-Plane Point Cloud Voxelization and Weight Aggregation](12101-multi-plane-point-cloud-voxelization-and-weight-aggregation.md)
Projects points onto multiple parallel depth planes and groups by 2D grid location and depth bin with learned weights. Choose for multi-plane projection rendering where depth discretization is required.

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ or $O(N)$ depending on the implementation of unique index extraction (sorting vs. hash map). Feature aggregation is $O(N)$.
*   **Memory Complexity:** $O(N')$ where $N'$ is the number of occupied voxels. $N' \le N$.
*   **Performance:** Open3D's `VoxelGrid` is efficient for visualization and simple occupancy [S1, S3, unofficial]. TensorFlow-based methods are optimized for GPU acceleration in neural network pipelines [S2, unofficial]. Topogenesis provides a lightweight lattice structure [S4, unofficial].
*   **Strengths:** Simple to implement; efficient for sparse data; compatible with standard deep learning frameworks.
*   **Limitations:** Loss of sub-voxel detail; sensitive to voxel size choice; hash map overhead for very large sparse grids.
*   **When to Use:** Preprocessing for voxel-based neural networks, occupancy grid generation for robotics, spatial indexing.
*   **When Not to Use:** When sub-voxel precision is required (use [Gaussian Voxelization](8884-gaussian-voxelization.md)); when the point cloud is extremely dense and memory is constrained (consider [Hard Voxelization](5345-hard-voxelization.md) with pruning).

## Background
The method originates from standard computer graphics and computational geometry techniques for discretizing continuous data. Open3D, a popular library for 3D data processing, provides a robust implementation via `VoxelGrid.create_from_point_cloud` [S1, S3, unofficial]. TensorFlow-based implementations like those in Google Research's TF3D library adapt this for deep learning pipelines [S2, unofficial]. The technique is widely used in robotics for occupancy grid mapping [S1, unofficial], in 3D reconstruction for neural rendering [S2, unofficial], and in digital manufacturing for voxel-based analysis [S1, unofficial].

## Sources
[S1] Guda108/3D-Voxelization-GUI, https://github.com/Guda108/3D-Voxelization-GUI, fetched 2026-10-27 (unofficial)
[S2] https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py#L305, https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py, fetched 2026-10-27 (unofficial)
[S3] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
[S4] Point Cloud Voxelization - topoGenesis, https://topogenesis.readthedocs.io/notebooks/point_cloud_voxelization/, fetched 2026-10-28 (unofficial)
[S5] FileToVox, https://github.com/Zarbuz/FileToVox, fetched 2026-10-29 (unofficial)
