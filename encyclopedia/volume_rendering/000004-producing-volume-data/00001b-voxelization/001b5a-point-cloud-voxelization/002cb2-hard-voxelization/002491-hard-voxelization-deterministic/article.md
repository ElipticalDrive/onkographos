# Hard Voxelization (Deterministic)

Also called: Point Cloud Voxelization via Histogramming, Voxel Grid Downsampling, Voxelization of unstructured 3D data

## Overview
Hard Voxelization (Deterministic) discretizes a continuous 3D point cloud into a fixed-size 3D grid of cubic voxels by binning points based on their spatial coordinates and aggregating associated attribute values (e.g., intensity, amplitude) within each voxel [S3, unofficial]. It outputs a single representative point per occupied voxel, calculated as the centroid of the aggregated attributes [S1, unofficial]. Unlike its sibling [Multiscale Hard Voxelization](3319-multiscale-hard-voxelization.md), which handles multiple resolutions, this method operates on a single fixed scale. It differs from the non-deterministic variant by guaranteeing reproducible output through explicit sorting and unique-coordinate identification, ensuring that the order of points within a voxel is preserved or consistently handled, whereas non-deterministic versions rely on atomic operations that may yield varying results across runs [S2, unofficial].

## How it works
The method maps continuous point coordinates $(x, y, z)$ to discrete integer voxel indices $(i, j, k)$ using floor division relative to a bounding box origin and voxel dimensions [S1, unofficial].

### Inputs and Outputs
**Inputs:**
- `points`: A 2D array of shape $(N, D)$, where $N$ is the number of points and $D$ is the number of features (typically 3 for coordinates, plus optional attributes like intensity) [S2, S3, unofficial].
- `voxel_size`: A 3-element array $[v_x, v_y, v_z]$ indicating the physical size of a voxel in x, y, and z dimensions [S2, unofficial].
- `coors_range`: A 6-element array $[x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$ defining the bounding box of the voxel grid [S2, unofficial].
- `max_points`: Integer. Maximum number of points allowed in a single voxel. If -1, dynamic voxelization is used, but the standard hard voxelization path typically enforces a limit [S2, unofficial].
- `max_voxels`: Integer. Maximum number of voxels to create. Points falling into voxels beyond this limit are dropped. Typical value is 20000 [S2, unofficial].

**Outputs:**
- `voxels`: A tensor of shape $[M, \text{max\_points}, D]$, where $M$ is the number of valid voxels ($\le \text{max\_voxels}$). It stores the features of the points assigned to each voxel [S2, unofficial].
- `coors`: A tensor of shape $[M, 3]$ containing the integer voxel indices $(i, j, k)$ for each of the $M$ voxels [S2, unofficial].
- `num_points_per_voxel`: A tensor of shape $[M]$ storing the number of points assigned to each voxel [S2, unofficial].

### Data Structures
1.  **Point**: Represents a single point with coordinates and attributes. In CPU implementations, this may be a struct with contiguous fields (e.g., `x, y, z, intensity, ring` as float32, 20 bytes total) [S1, unofficial]. In GPU implementations, points are stored in flat arrays [S2, unofficial].
2.  **Voxel Grid (CPU)**: Two 3D arrays `vcnt` and `vsum` of dimensions $(X, Y, Z)$, where $X = \lfloor \frac{x_{max} - x_{min}}{v_x} \rfloor + 1$, etc. `vcnt` stores the count of points, and `vsum` stores the sum of attribute values for each voxel [S3, unofficial].
3.  **Voxel Grid (GPU)**: A linear array or hash map mapping linear voxel indices to a `Voxel` struct containing sums (`sum_x`, `sum_y`, `sum_z`, `sum_intensity`, `sum_ring`) and a `count` integer [S1, unofficial].

### Steps
1.  **Compute Bounding Box and Grid Dimensions**:
    -   If `coors_range` is not provided, iterate over all input points to find $x_{min}, x_{max}, y_{min}, y_{max}, z_{min}, z_{max}$ [S1, unofficial].
    -   Calculate grid dimensions using ceiling division:
        $$
        \text{dim}_x = \max(1, \lceil \frac{x_{max} - x_{min}}{v_x} \rceil), \quad
        \text{dim}_y = \max(1, \lceil \frac{y_{max} - y_{min}}{v_y} \rceil), \quad
        \text{dim}_z = \max(1, \lceil \frac{z_{max} - z_{min}}{v_z} \rceil)
        $$
        [S1, unofficial]
    -   Ensure dimensions are at least 1 to prevent zero-size allocations [S1, unofficial].

2.  **Map Points to Voxel Indices**:
    -   For each point $p = [x, y, z, \dots]$, compute voxel coordinates:
        $$
        i = \left\lfloor \frac{x - x_{min}}{v_x} \right\rfloor, \quad
        j = \left\lfloor \frac{y - y_{min}}{v_y} \right\rfloor, \quad
        k = \left\lfloor \frac{z - z_{min}}{v_z} \right\rfloor
        $$
        [S1, S2, S3, unofficial]
    -   Points with coordinates outside the range $R$ are mapped to invalid indices (e.g., $-1$) [S1, S2, unofficial].

3.  **Identify Unique Voxels and Map Indices (Deterministic Step)**:
    -   Collect all valid voxel coordinates $(i, j, k)$ (excluding $-1$) [S2, unofficial].
    -   Find unique voxel coordinates and generate a mapping from the original point index to a unique voxel index. This is typically done using a stable sort or a unique operation (e.g., `at::unique_dim` in PyTorch) to ensure reproducibility [S2, unofficial].
    -   If the invalid coordinate $(-1, -1, -1)$ is present in the unique list, remove it and adjust the mapping indices accordingly [S2, unofficial].
    -   Compute `coors_order` (the order in which voxels appear in the output) and `reduce_count` (number of points per voxel) [S2, unofficial].

4.  **Aggregate Attributes**:
    -   **CPU Approach**: Iterate through all points. For each point, increment `vcnt[i, j, k]` and add its attributes to `vsum[i, j, k]` [S3, unofficial].
    -   **GPU Approach**: Use a parallel kernel to accumulate sums into a `Voxel` struct array. Shared memory may be used for block-level reduction [S1, unofficial].
    -   Compute the average attribute value per voxel:
        $$
        \text{vavg} = \frac{\text{vsum}}{\text{vcnt}}
        $$
        [S3, unofficial]
    -   Handle empty voxels by setting their average value to NaN where the count is less than 1: `vavg[vcnt < 1] = np.nan` [S3, unofficial].

5.  **Assign Features to Output Tensors**:
    -   Iterate over the voxel grid or use a parallel kernel to write the aggregated data into the `voxels`, `coors`, and `num_points_per_voxel` tensors [S2, unofficial].
    -   For each voxel, store the coordinates in `coors` and the count in `num_points_per_voxel` [S2, unofficial].
    -   Store the point features (or centroids) in `voxels`. If multiple points exist in a voxel, they are stored sequentially up to `max_points` [S2, unofficial].
    -   Clamp the number of points per voxel to `max_points` [S2, unofficial].

### Pseudocode (CPU Implementation)
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def hard_voxelize_deterministic(points, voxel_size, coors_range, max_points=100, max_voxels=20000):
    """
    points: (N, D) array, D >= 3
    voxel_size: [vx, vy, vz]
    coors_range: [xmin, ymin, zmin, xmax, ymax, zmax]
    """
    x_min, y_min, z_min, x_max, y_max, z_max = coors_range
    vx, vy, vz = voxel_size
    
    # Step 1: Compute grid dimensions
    dim_x = max(1, int(np.ceil((x_max - x_min) / vx)))
    dim_y = max(1, int(np.ceil((y_max - y_min) / vy)))
    dim_z = max(1, int(np.ceil((z_max - z_min) / vz)))
    
    # Step 2: Map points to voxel indices
    # Calculate indices for all points
    xi = np.floor((points[:, 0] - x_min) / vx).astype(np.int32)
    yi = np.floor((points[:, 1] - y_min) / vy).astype(np.int32)
    zi = np.floor((points[:, 2] - z_min) / vz).astype(np.int32)
    
    # Identify valid points (within range)
    valid_mask = (xi >= 0) & (xi < dim_x) & (yi >= 0) & (yi < dim_y) & (zi >= 0) & (zi < dim_z)
    valid_points = points[valid_mask]
    valid_xi = xi[valid_mask]
    valid_yi = yi[valid_mask]
    valid_zi = zi[valid_mask]
    
    if len(valid_points) == 0:
        return np.empty((0, max_points, points.shape[1])), np.empty((0, 3)), np.empty(0)
    
    # Step 3: Identify unique voxels and map indices
    # Combine coordinates into a single linear index or tuple for sorting
    # Using linear index for efficiency: idx = i + j*dim_x + k*dim_x*dim_y
    linear_indices = valid_xi + valid_yi * dim_x + valid_zi * dim_x * dim_y
    
    # Sort by linear index to group points by voxel
    sort_order = np.argsort(linear_indices)
    sorted_linear_indices = linear_indices[sort_order]
    sorted_points = valid_points[sort_order]
    
    # Find unique voxels and their counts
    unique_voxels, counts = np.unique(sorted_linear_indices, return_counts=True)
    
    # Limit to max_voxels
    if len(unique_voxels) > max_voxels:
        unique_voxels = unique_voxels[:max_voxels]
        counts = counts[:max_voxels]
        
    # Create a mapping from linear index to voxel output index
    voxel_to_output_idx = {v: i for i, v in enumerate(unique_voxels)}
    
    # Step 4: Aggregate attributes
    # Initialize output structures
    num_valid_voxels = len(unique_voxels)
    coors = np.zeros((num_valid_voxels, 3), dtype=np.int32)
    num_points_per_voxel = np.zeros(num_valid_voxels, dtype=np.int32)
    voxels = np.zeros((num_valid_voxels, max_points, points.shape[1]), dtype=points.dtype)
    
    # Re-iterate to fill voxels (deterministic order due to sorting)
    # We need to map back to the original sorted points
    # Since we sorted, we can iterate through the sorted points and assign them to the correct voxel bucket
    
    current_voxel_idx = 0
    current_voxel_linear = unique_voxels[0]
    point_count_in_voxel = 0
    
    # Helper to convert linear index back to (i, j, k)
    def linear_to_coor(idx):
        k = idx // (dim_x * dim_y)
        rem = idx % (dim_x * dim_y)
        j = rem // dim_x
        i = rem % dim_x
        return np.array([i, j, k])

    for idx, linear_idx in enumerate(sorted_linear_indices):
        # If we moved to a new voxel
        if linear_idx != current_voxel_linear:
            current_voxel_linear = linear_idx
            current_voxel_idx = voxel_to_output_idx[linear_idx]
            point_count_in_voxel = 0
            coors[current_voxel_idx] = linear_to_coor(linear_idx)
            
        # Assign point to voxel
        if point_count_in_voxel < max_points:
            voxels[current_voxel_idx, point_count_in_voxel, :] = sorted_points[idx]
        point_count_in_voxel += 1
        
    # Update num_points_per_voxel
    # We need to count how many points actually went into each voxel
    # Since we iterated in order, we can just count based on the unique counts, capped by max_points
    for i, count in enumerate(counts):
        num_points_per_voxel[i] = min(count, max_points)
        
    return voxels, coors, num_points_per_voxel
```

### Parameters and Typical Values
-   `voxel_size`: Typically $[0.05, 0.05, 0.1]$ meters for autonomous driving scenes [S1, S2, unofficial].
-   `coors_range`: Defined by the sensor's field of view, e.g., $[-40, -40, -3, 40, 40, 1]$ for nuScenes [S1, S2, unofficial].
-   `max_points`: Typically 10-30 points per voxel [S2, unofficial].
-   `max_voxels`: Typically 20000-40000 voxels [S2, unofficial].

### Edge Cases and Pitfalls
-   **Points Outside Range**: Points with coordinates outside `coors_range` are mapped to $-1$ and must be filtered out before unique operations to avoid invalid indices [S2, unofficial].
-   **Empty Voxels**: Voxels with no points should be excluded from the output or marked with NaN for attributes to avoid division by zero during averaging [S3, unofficial].
-   **Boundary Precision**: Floating-point precision issues may cause points exactly on the boundary to be assigned to the wrong voxel. Clamping indices to $[0, \text{dim}-1]$ is recommended [S1, unofficial].
-   **Coordinate Systems**: Ensure the input point cloud coordinate system matches the expected range. For example, nuScenes uses $x=$right, $y=$forward, $z=$up, which may need conversion to KITTI/ROS ($x=$forward, $y=$left, $z=$up) [S1, unofficial].

## Subtopics

### [Fast Deterministic GPU Voxelization (Sort-Scatter)](3806-fast-deterministic-gpu-voxelization-sort-scatter.md)
Fast Deterministic GPU Voxelization is a PyTorch-based method for converting 3D point clouds into a fixed-size voxel grid that achieves bit-identical output to a reference C++/CUDA implementation while providing a ~100x speedup. It replaces an $O(N^2)$ sequential CUDA kernel with a parallel pipeline of filtering, coordinate computation, stable sorting, and scattering. Unlike the standard `deterministic=False` path which relies on non-reproducible atomic operations, this method guarantees reproducibility by preserving original point order within voxels via stable sorting. Choose this when bit-identical reproducibility is required on GPU. Link: [Fast Deterministic GPU Voxelization (Sort-Scatter)](3806-fast-deterministic-gpu-voxelization-sort-scatter.md).

### [Grid Pooling (Max Pooling over Points)](4327-grid-pooling-max-pooling-over-points.md)
Grid Pooling aggregates point cloud features into a fixed-size 3D voxel grid by performing max-pooling within each grid cell. It maps a set of $N$ points with $C$-dimensional features to a $W \times H \times D$ grid where each cell stores the maximum feature value among points falling into that cell. It differs from its sibling [Hard Voxelization (Deterministic)](3321-hard-voxelization-deterministic.md) by explicitly tracking the index of the point that achieved the maximum value to enable differentiable backpropagation, whereas hard voxelization typically discards this linkage or handles duplicates via simple overwriting or averaging without gradient tracking. Choose this when differentiable aggregation is needed. Link: [Grid Pooling (Max Pooling over Points)](4327-grid-pooling-max-pooling-over-points.md).

### [Grid-based Scene Cropping with KNN Precomputation](4462-grid-based-scene-cropping-with-knn-precomputation.md)
This method generates cropped point cloud datasets by dividing a normalized 3D scene into a structured grid, applying morphological dilation to expand active voxel regions, and precomputing K-nearest neighbor (KNN) relationships for both input points and grid voxels. It solves the problem of efficiently preparing local feature grouping inputs for deep learning models by providing fixed-size, spatially coherent data structures that include relative coordinate offsets. It differs from its siblings by explicitly precomputing KNN indices and relative coordinates for both points and voxels within a cropped region, whereas [Hard Voxelization (Deterministic)](3321-hard-voxelization-deterministic.md) focuses on static assignment and [Grid Pooling (Max Pooling over Points)](4327-grid-pooling-max-pooling-over-points.md) focuses on aggregation rather than neighbor indexing. Choose this for preparing local feature grouping inputs. Link: [Grid-based Scene Cropping with KNN Precomputation](4462-grid-based-scene-cropping-with-knn-precomputation.md).

### Hard Voxelization (Non-Deterministic)
A faster, non-reproducible variant of hard voxelization that uses atomic operations or parallel reductions to assign points to voxels, avoiding the sorting/unique step of the deterministic version [S2, unofficial]. It filters out invalid coordinates, finds unique voxel coordinates using `at::unique_dim`, and then uses `nondisterministic_get_assign_pos` and `nondisterministic_assign_point_voxel` kernels to assign features. It is faster but not reproducible. Choose this when speed is prioritized over reproducibility.

### Nearest Neighbour Interpolation for Voxelization
A method to assign attribute values to voxel centers by finding the closest point in the original point cloud to each voxel center using a KDTree, rather than averaging points within the voxel volume [S3, unofficial]. It constructs a KDTree, queries the nearest neighbor for each voxel center, and filters out points that are too far away (distance $\ge$ half the voxel width) by setting their attribute values to NaN [S3, unofficial]. Unlike histogramming which averages all points within a voxel, this method assigns the value of the single closest point. Choose this when a specific point's value is preferred over an average.

## Operating characteristics
-   **Time Complexity**: $O(N \log N)$ due to sorting or unique operations on $N$ points [S2, unofficial]. The GPU parallel version can achieve significant speedups (~100x) over sequential CPU implementations [S1, unofficial].
-   **Memory Complexity**: $O(M \cdot \text{max\_points} \cdot D)$ for the output tensors, where $M$ is the number of voxels. The grid storage is $O(\text{dim}_x \cdot \text{dim}_y \cdot \text{dim}_z)$ if a dense grid is used [S1, S3, unofficial].
-   **Measured Performance**: On the nuScenes dataset, the parallel CUDA version significantly outperforms the sequential CPU version, with execution times logged in CSV files for benchmarking [S1, unofficial].
-   **Strengths**: Deterministic output, simple implementation, efficient downsampling.
-   **Limitations**: Fixed resolution, loss of information within voxels (averaging), sensitivity to voxel size and range.
-   **When to Use**: When a fixed-resolution representation is required for deep learning models (e.g., VoxelNet, SECOND) and reproducibility is important.
-   **When Not to Use**: When variable resolution is needed (use [Multiscale Hard Voxelization](3319-multiscale-hard-voxelization.md)) or when differentiable aggregation is required (use [Grid Pooling](4327-grid-pooling-max-pooling-over-points.md)).

## Background
Hard Voxelization is a fundamental technique in point cloud processing, particularly in autonomous driving and 3D computer vision. It was popularized by methods like VoxelNet and SECOND, which use voxelized point clouds as input to 3D convolutional neural networks [S1, S2, unofficial]. The deterministic variant ensures that the downsampling process is reproducible, which is crucial for debugging and benchmarking [S2, unofficial]. The method is widely used in datasets like nuScenes and KITTI for preparing point cloud data for 3D object detection [S1, unofficial].

## Sources
[S1] Aleyna-B/Point-Cloud-Voxelization, https://github.com/Aleyna-B/Point-Cloud-Voxelization, fetched 2026-10-27 (unofficial)
[S2] Susie0909/Multiscale_Voxelization_CUDA, https://github.com/Susie0909/Multiscale_Voxelization_CUDA, fetched 2026-10-27 (unofficial)
[S3] Point Cloud Voxelization with Python (numpy & scipy), https://medium.com/@h.shaig93/voxelization-with-python-numpy-scipy-b7f013890da6, fetched 2026-10-28 (unofficial)
