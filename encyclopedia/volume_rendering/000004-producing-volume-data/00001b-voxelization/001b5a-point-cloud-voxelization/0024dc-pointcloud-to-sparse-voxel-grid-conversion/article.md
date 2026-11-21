# Pointcloud to Sparse Voxel Grid Conversion

Also called: Voxelize Point Cloud

## Overview
Pointcloud to Sparse Voxel Grid Conversion maps a set of unstructured 3D points into a discrete, sparse voxel grid representation by assigning each point to a specific voxel index within a defined bounding box [S1, S3, S4, unofficial]. It solves the problem of organizing irregular point data into a structured format suitable for downstream rendering, reconstruction, or spatial analysis [S1, unofficial]. Unlike direct occupancy mapping which may store redundant data, this method aggregates features or simply marks occupancy, differing from [DCX Points-to-Voxels](3783-dcx-points-to-voxels.md) by prioritizing depth confidence in aggregation, and from [VoxelGrid::CreateFromPointCloudWithinBounds](8505-voxelgrid-createfrompointcloudwithinbounds.md) by using structured spatial partitioning rather than a direct hash-map accumulation [S1, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A point cloud consisting of $N$ points, where each point has 3D coordinates $(x, y, z)$. Optional attributes include color, normals, or feature vectors [S1, S3, S4, unofficial].
*   **Parameters:**
    *   `voxel_size` or `spacing`: The side length of a voxel cube. Default is often 0.05 or {1, 1, 1} depending on the implementation [S1, S2, S5, unofficial]. Must be $> 0$ [S2, unofficial].
    *   `downsample_size`: Optional initial voxel downsampling size (e.g., 0.03) to reduce point count before conversion [S1, unofficial].
    *   `padding`: Relative bounding box expansion factor (e.g., 0.1% or 0.001) to prevent points from landing exactly on grid boundaries [S2, unofficial].
*   **Output:** A sparse voxel grid containing occupied voxel indices and optionally aggregated features (e.g., centroid coordinates, mean color) [S1, S3, unofficial]. Output formats include CSV (x, y, z, voxel_size), NPY (dictionary with centers, voxel_size, count), or internal tensor structures [S1, S3, unofficial].

**Data Structures:**
*   **Voxel Indices:** 3D integer arrays or flattened 1D indices mapping points to voxels [S3, unofficial].
*   **Sparse Storage:** Only voxels containing points are stored, often using hash maps or sorted arrays to save memory [S3, S4, unofficial].
*   **Features:** If features are present, they are aggregated per voxel using functions like mean, max, or min [S3, unofficial].

**Steps:**
1.  **Load Point Cloud:** Read input file (e.g., .ply, .pcd, .xyz, .csv) into memory. For CSV/TXT, extract the first three numeric columns as x, y, z [S1, S4, unofficial]. Validate that the vertex list has 3 components [S2, unofficial].
2.  **Preprocessing (Optional):**
    *   **Downsampling:** If enabled, apply voxel grid downsampling to reduce point density [S1, unofficial].
    *   **Outlier Removal:** Remove statistical outliers using a neighbor count (e.g., 20) and standard deviation ratio (e.g., 2.0) [S1, unofficial].
3.  **Compute Bounding Box:** Calculate the minimum and maximum coordinates of the point cloud. Apply padding to the extent to ensure no points lie on the boundary [S2, unofficial].
    *   $extent = max - min$
    *   $padded\_extent = extent \times (1 + padding)$
4.  **Compute Grid Dimensions:** Calculate the number of voxels along each axis.
    *   $dims_i = \lceil padded\_extent_i / spacing_i \rceil$
    *   Clamp each dimension to a minimum of 1 to handle planar or collinear clouds [S2, unofficial].
5.  **Map Points to Voxels:** For each point $P_j = (x_j, y_j, z_j)$:
    *   Compute raw grid coordinate: $x_{raw} = (x_j - origin_x) \times (1.0 / spacing_x)$
    *   Validate $x_{raw}$ in float domain: if $x_{raw} < 0$ or $x_{raw} \ge dims_x$, skip point [S2, unofficial].
    *   Cast to integer index: $x_{idx} = \lfloor x_{raw} \rfloor$
    *   Repeat for y and z axes [S2, S3, unofficial].
6.  **Aggregate Features (Optional):** If features are present, group points by voxel index and apply an aggregation function (e.g., mean, max) [S3, unofficial].
    *   $voxel\_features = \text{segment\_func}(features, voxel\_indices)$
7.  **Extract Centers:** For each occupied voxel, compute the center coordinate:
    *   $center = origin + (voxel\_index + 0.5) \times voxel\_size$ [S1, unofficial].
8.  **Output:** Export the list of voxel centers and associated data [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def pointcloud_to_sparse_voxel_grid(points, voxel_size, padding=0.001):
    """
    Convert a point cloud to a sparse voxel grid.
    
    Args:
        points: numpy array of shape (N, 3) containing x, y, z coordinates.
        voxel_size: float, size of each voxel cube.
        padding: float, relative padding for bounding box (default 0.1%).
        
    Returns:
        centers: numpy array of shape (M, 3) containing voxel center coordinates.
        voxel_size: float, the voxel size used.
    """
    if points.size == 0:
        raise ValueError("Point cloud is empty.")
    
    # Step 1: Compute bounding box
    min_coords = np.min(points, axis=0)
    max_coords = np.max(points, axis=0)
    extent = max_coords - min_coords
    
    # Step 2: Apply padding
    padded_extent = extent * (1 + padding)
    
    # Step 3: Compute grid dimensions
    dims = np.ceil(padded_extent / voxel_size).astype(int)
    dims = np.maximum(dims, 1) # Clamp to minimum 1
    
    # Step 4: Map points to voxel indices
    # Calculate origin as the min_coords
    origin = min_coords
    
    # Compute raw grid coordinates
    # Avoid division by zero if voxel_size is 0 (though validated elsewhere)
    if voxel_size <= 0:
        raise ValueError("Voxel size must be positive.")
        
    grid_coords = (points - origin) / voxel_size
    
    # Validate and cast to integer indices
    # Check bounds in float domain to handle NaN/Inf
    valid_mask = (grid_coords >= 0) & (grid_coords < dims)
    valid_mask = np.all(valid_mask, axis=1)
    
    if not np.any(valid_mask):
        raise ValueError("No points fall within the voxel grid bounds.")
        
    valid_points = points[valid_mask]
    valid_grid_coords = grid_coords[valid_mask]
    
    # Cast to integer indices
    voxel_indices = np.floor(valid_grid_coords).astype(int)
    
    # Step 5: Extract unique occupied voxels and compute centers
    # Get unique voxel indices
    unique_indices, inverse_indices = np.unique(voxel_indices, axis=0, return_inverse=True)
    
    # Compute centers: origin + (index + 0.5) * voxel_size
    centers = (unique_indices + 0.5) * voxel_size + origin
    
    return centers, voxel_size
```

**Edge Cases and Pitfalls:**
*   **Precision Collapse:** For distant small clouds, float32 precision may cause padding to be negligible, leading to zero extent. Use float64 for bounding box arithmetic or add an absolute epsilon floor [S2, unofficial].
*   **Out-of-Range Points:** Points outside the bounding box are silently dropped in some implementations. Emit a warning if points are skipped [S2, unofficial].
*   **Empty or Degenerate Input:** Handle empty point clouds or planar/collinear points by clamping dimensions to 1 [S2, unofficial].
*   **Integer Overflow:** Validate that the product of dimensions does not overflow the output array size [S2, unofficial].

## Subtopics

### [DCX Points-to-Voxels](3783-dcx-points-to-voxels.md)
DCX Points-to-Voxels converts a point cloud into a sparse voxel grid by mapping points to voxel indices and aggregating features, prioritizing points with higher depth confidence to mitigate noise. It serves as the final aggregation step in the DCX pipeline, contrasting with direct occupancy mapping.

### [VoxelGrid::CreateFromPointCloudWithinBounds](8505-voxelgrid-createfrompointcloudwithinbounds.md)
`VoxelGrid::CreateFromPointCloudWithinBounds` voxalizes a point cloud into a sparse voxel grid using a direct hash-map accumulation approach based on a fixed voxel size and minimum bounding box corner, ignoring the maximum bound for coordinate calculation. It is significantly faster than mesh-based voxelization variants.

### [Multi-Plane Point Cloud Voxelization and Weight Aggregation](12101-multi-plane-point-cloud-voxelization-and-weight-aggregation.md)
A pre-processing pipeline that projects a 3D point cloud onto multiple parallel planes based on depth, groups points by 2D grid location and depth bin, and computes aggregation weights for rendering. It reduces outliers and blends points front-to-back using a visibility mask, avoiding z-fighting issues.

### Point Cloud I/O
Reads and writes point cloud data in PLY format, extracting positions, colors, and normals. PLY vertex fields include x, y, z (f4), nx, ny, nz (f4), and red, green, blue (u1) [S7, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ for point cloud scan and mapping, where $N$ is the number of points. Grid dimension computation is $O(1)$ or $O(dims)$ depending on implementation [S2, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory excluding output, $O(M)$ for output mask/grid where $M$ is the number of occupied voxels [S2, unofficial].
*   **Performance:** Depends on hardware and resolution. Out-of-core implementations may suffer from I/O thrashing if random-order scatter writes are not buffered [S2, unofficial].
*   **Strengths:** Efficiently handles sparse data, supports various input formats, and provides structured output for rendering [S1, unofficial].
*   **Limitations:** Sensitive to voxel size selection; small voxels lead to large grids, large voxels lose detail. Precision issues can arise with distant clouds [S2, unofficial].

## Background
Originating from the need to discretize 3D geometric data for spatial analysis and rendering, this method is widely used in robotics, additive manufacturing, and computational geometry [S1, unofficial]. Tools like the 3D-Voxelization-GUI by Guda108 provide accessible implementations for converting point clouds and meshes into voxel centers [S1, unofficial]. The technique has evolved to include advanced aggregation methods and out-of-core processing capabilities to handle large-scale point clouds [S2, S3, unofficial].

## Sources
[S1] Guda108/3D-Voxelization-GUI, https://github.com/Guda108/3D-Voxelization-GUI, fetched 2026-10-27 (unofficial)
[S2] FILT: Voxelize Point Cloud, https://github.com/BlueQuartzSoftware/simplnx/pull/1724, fetched 2026-10-27 (unofficial)
[S3] https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py#L305, https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py, fetched 2026-10-27 (unofficial)
[S4] How to Voxelize Meshes and Point Clouds in Python - Medium, https://medium.com/data-science/how-to-voxelize-meshes-and-point-clouds-in-python-ca94d403f81d, fetched 2026-10-28 (unofficial)
[S5] Point Cloud Voxelization - topoGenesis, https://topogenesis.readthedocs.io/notebooks/point_cloud_voxelization/, fetched 2026-10-28 (unofficial)
[S6] FileToVox, https://github.com/Zarbuz/FileToVox, fetched 2026-10-29 (unofficial)
[S7] minghanqin/LangSplat, https://github.com/minghanqin/LangSplat, fetched 2026-10-27 (unofficial)
