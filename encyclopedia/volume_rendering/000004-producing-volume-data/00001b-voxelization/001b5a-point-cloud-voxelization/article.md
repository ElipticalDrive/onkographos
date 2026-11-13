# Point Cloud Voxelization

Also called: Pointcloud to Dense Voxel Grid Conversion, Voxelization from Point Cloud

## Overview
Point Cloud Voxelization converts an unstructured set of 3D points into a discrete volumetric grid representation by mapping each point to a specific voxel index based on spatial coordinates [S2][S3, S6, unofficial]. It solves the problem of transforming sparse, irregular point data into a regular tensor structure suitable for convolutional neural networks, spatial queries, or dense volume rendering [S3, S6, unofficial]. Unlike [Hard Voxelization](5345-hard-voxelization.md), which often implies binary occupancy or simple binning for deep learning pipelines, this method explicitly supports feature aggregation (e.g., mean, max) and sparse storage formats to handle varying point densities and attribute dimensions [S2][S6, unofficial]. It differs from [Ray Model Voxelization](4688-ray-model-voxelization.md) by directly binning observed points rather than inferring occupancy from sensor rays [S4, unofficial].

## How it works
**Inputs:**
1.  `points`: Array of shape $(N, D)$, where $N$ is the number of points and $D \ge 3$. The first three columns are coordinates $(x, y, z)$. Remaining columns are features.
2.  `voxel_size` or `grid_cell_size`: Array of shape $(3,)$ defining the physical dimensions of a voxel $(\Delta x, \Delta y, \Delta z)$.
3.  `grid_range` or `start_location`/`end_location`: Defines the spatial bounds. `grid_range` is $(min_x, min_y, min_z, max_x, max_y, max_z)$ [S3, unofficial]. `start_location` is $(min_x, min_y, min_z)$ and `end_location` is $(max_x, max_y, max_z)$ [S6, unofficial].
4.  `segment_func` (optional): Aggregation function for features (e.g., mean, sum, max) [S6, unofficial].
5.  `max_num_voxels` (optional): Hard limit on the number of occupied voxels in sparse formats [S3, unofficial].
6.  `max_points_in_voxel` (optional): Hard limit on points stored per voxel in sparse formats [S3, unofficial].

**Outputs:**
*   **Dense Format:** A 4D tensor of shape $(N_x, N_y, N_z, F)$ containing aggregated voxel features [S6, unofficial].
*   **Sparse Format:** A tuple of arrays:
    *   `voxels`: Shape $(V, M, D)$, where $V$ is the number of active voxels, $M$ is `max_points_in_voxel`, and $D$ is the feature dimension.
    *   `coordinates`: Shape $(V, 3)$, the grid indices of active voxels.
    *   `num_points_per_voxel`: Shape $(V,)$, the count of points in each active voxel [S3, unofficial].

**Data Structures:**
*   **Coordinate-to-Index Map (`coor_to_voxelidx`):** A 3D array of shape $(N_z, N_y, N_x)$ with dtype `int32`, initialized to $-1$. It maps grid coordinates to a sparse linear index [S3, unofficial].
*   **Voxel Buffer (`voxels`):** Pre-allocated array of shape $(max\_num\_voxels, max\_points\_in\_voxel, D)$ [S3, unofficial].

**Steps:**
1.  **Grid Dimension Calculation:**
    Compute the number of voxels along each axis.
    $$ N_x = \lfloor \frac{max\_x - min\_x}{\Delta x} \rfloor, \quad N_y = \lfloor \frac{max\_y - min\_y}{\Delta y} \rfloor, \quad N_z = \lfloor \frac{max\_z - min\_z}{\Delta z} \rfloor $$
    Using `grid_range` and `voxel_size` [S3, unofficial]:
    $$ grid\_size = \lfloor (grid\_range[3:] - grid\_range[:3]) / voxel\_size \rfloor $$
    Using `start_location` and `end_location` [S6, unofficial]:
    $$ grid\_size = \lceil (end\_location - start\_location) / grid\_cell\_size \rceil $$
    [S3, S6, unofficial]

2.  **Coordinate Discretization:**
    Map continuous point coordinates to integer grid indices.
    $$ idx = \lfloor \frac{p - start\_location}{voxel\_size} \rfloor $$
    Cast to `int32` [S3, S6, unofficial].

3.  **Boundary Handling:**
    *   **Discard (Sparse):** Create a mask where $0 \le idx < grid\_size$ for all dimensions. Discard points failing this mask [S3, unofficial].
    *   **Clamp (Dense):** Clamp indices to $[0, grid\_size - 1]$. Points outside bounds are assigned to the nearest edge voxel [S6, unofficial].

4.  **Feature Aggregation (Dense):**
    *   Compute 1D segment IDs for each point:
        $$ segment\_id = idx_x \cdot (N_y N_z) + idx_y \cdot N_z + idx_z $$
    *   Apply `segment_func` (default `unsorted_segment_mean`) to aggregate features from `points` into a flattened tensor, then reshape to $(N_x, N_y, N_z, F)$ [S6, unofficial].

5.  **Sparse Population:**
    *   Initialize `coor_to_voxelidx` with $-1$, `voxels` with zeros, and `num_points_per_voxel` with zeros.
    *   Iterate through valid points. For point $i$ with coordinate $c$:
        *   Lookup `voxel_id = coor_to_voxelidx[c]`.
        *   If `voxel_id == -1`:
            *   If `voxel_num >= max_num_voxels`, break (discard remaining points) [S3, unofficial].
            *   Set `voxel_id = voxel_num`, increment `voxel_num`.
            *   Set `coor_to_voxelidx[c] = voxel_id`.
            *   Store $c$ in `coordinates[voxel_id]`.
        *   If `num_points_per_voxel[voxel_id] < max_points_in_voxel`:
            *   Store point $i$ in `voxels[voxel_id, num_points_per_voxel[voxel_id]]`.
            *   Increment `num_points_per_voxel[voxel_id]` [S3, unofficial].

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
    points_copy = points.copy()
    # 1. Grid Size
    grid_size = np.floor((grid_range[3:] - grid_range[:3]) / voxel_size).astype(np.int32)
    
    # 2. Init Structures
    # Shape is (Z, Y, X) to match typical memory layout or explicit indexing
    coor_to_voxelidx = np.full((grid_size[2], grid_size[1], grid_size[0]), -1, dtype=np.int32)
    voxels = np.zeros((max_num_voxels, max_points_in_voxel, points_copy.shape[-1]), dtype=points_copy.dtype)
    coors = np.zeros((max_num_voxels, 3), dtype=np.int32)
    num_points_per_voxel = np.zeros(max_num_voxels, dtype=np.int32)
    
    # 3. Discretize
    coor = np.floor((points_copy[:, :3] - grid_range[:3]) / voxel_size).astype(np.int32)
    
    # 4. Mask Valid Points
    mask = (
        (coor[:, 0] >= 0) & (coor[:, 0] < grid_size[0]) &
        (coor[:, 1] >= 0) & (coor[:, 1] < grid_size[1]) &
        (coor[:, 2] >= 0) & (coor[:, 2] < grid_size[2])
    )
    coor = coor[mask]
    points_copy = points_copy[mask]
    
    voxel_num = 0
    # 5. Populate
    for i, c in enumerate(coor):
        # c is (x, y, z)
        voxel_id = coor_to_voxelidx[c[2], c[1], c[0]] # Note: array dims are Z,Y,X
        
        if voxel_id == -1:
            if voxel_num >= max_num_voxels:
                break
            voxel_id = voxel_num
            voxel_num += 1
            coor_to_voxelidx[c[2], c[1], c[0]] = voxel_id
            coors[voxel_id] = c
            
        n_pts = num_points_per_voxel[voxel_id]
        if n_pts < max_points_in_voxel:
            voxels[voxel_id, n_pts] = points_copy[i]
            num_points_per_voxel[voxel_id] += 1
            
    return voxels[:voxel_num], coors[:voxel_num], num_points_per_voxel[:voxel_num]
```

## Subtopics

### [Octree Construction for Point Clouds](1070-octree-construction-for-point-clouds.md)
Constructs a hierarchical octree for variable-resolution storage, chosen when point density varies significantly across the scene.

### [Point Resampling with Neighbor Masking](2710-point-resampling-with-neighbor-masking.md)
Reduces point cloud density before voxelization by masking neighbors of active regions, chosen for compression while preserving structural fidelity.

### [Voxels to Points Projection](3172-voxels-to-points-projection.md)
Reverses the process by projecting voxel features back to original points, chosen for feature transfer between dense and sparse representations.

### [Pointcloud to Sparse Voxel Grid Conversion](3416-pointcloud-to-sparse-voxel-grid-conversion.md)
A specific implementation focusing on sparse storage with feature aggregation, chosen for memory efficiency in large scenes.

### [Point Sampling from Mesh](3786-point-sampling-from-mesh.md)
Generates the input point cloud from a mesh surface, chosen when the source is a triangle mesh rather than raw points.

### [Ray Model Voxelization](4688-ray-model-voxelization.md)
Uses sensor rays to classify voxels as free/occupied, chosen for LiDAR data where empty space information is critical.

### [Two-stage training with adaptive point cloud generation](4714-two-stage-training-with-adaptive-point-cloud-generation.md)
Generates supplemental points to refine sparse representations, chosen for iterative neural field training.

### [Hard Voxelization](5345-hard-voxelization.md)
Strict binning for CNN input, chosen for standard 3D object detection pipelines.

### [Voxel-based Void Ratio Computation](6679-voxel-based-void-ratio-computation.md)
Computes physical void ratios via sphere-box integration, chosen for geotechnical particle simulations.

### [Point Cloud Cropping and Downsampling](7057-point-cloud-cropping-and-downsampling.md)
Pre-processes points by cropping and random sampling, chosen to reduce noise and size before voxelization.

### [Gaussian Voxelization](8884-gaussian-voxelization.md)
Converts Gaussian primitives to voxels, chosen for 3D Gaussian Splatting rendering pipelines.

## Operating characteristics
**Time Complexity:** $O(N)$ for coordinate mapping and masking. Sparse population is $O(V \cdot M)$ in worst case but typically $O(N)$ if $M$ is small and hash/array lookup is $O(1)$. Dense aggregation is $O(N)$ using `unsorted_segment_*` operations [S3, S6, unofficial].
**Memory Complexity:**
*   Dense: $O(N_x N_y N_z F)$. Can be prohibitive for large volumes with low occupancy.
*   Sparse: $O(V \cdot M \cdot D + V \cdot 3)$. Requires pre-allocation of `max_num_voxels` and `max_points_in_voxel` buffers [S3, unofficial].
**Performance:** JIT-compiled versions (e.g., Numba) significantly reduce Python overhead, achieving real-time performance for LiDAR point clouds [S3, unofficial].
**Strengths:** Simple implementation; supports arbitrary feature dimensions; sparse format handles large, empty scenes efficiently.
**Limitations:** Fixed resolution limits detail; boundary handling (discard vs. clamp) affects data integrity; sparse format requires careful management of capacity limits (`max_num_voxels`) which may truncate data if underestimated [S3, unofficial].

## Background
Point cloud voxelization is a fundamental preprocessing step in 3D computer vision, particularly for LiDAR processing and 3D deep learning. Early implementations focused on binary occupancy grids for robot navigation [S4, unofficial]. With the rise of 3D CNNs, methods evolved to support feature aggregation (mean, max) to preserve semantic information [S6, unofficial]. Libraries like Open3D provide high-level APIs for mesh and point cloud voxelization [S2]. Sparse voxelization techniques, often used in autonomous driving perception stacks, optimize memory by storing only occupied voxels and their associated points [S3, unofficial].

## Sources
[S1] autonomousvision/differentiable_volumetric_rendering, https://github.com/autonomousvision/differentiable_volumetric_rendering, fetched 2026-10-26 (unofficial)
[S2] Voxelization — Open3D 0.12.0 documentation, https://www.open3d.org/docs/0.12.0/tutorial/geometry/voxelization.html, fetched 2026-10-26
[S3] danielhavir/voxelize3d, https://github.com/danielhavir/voxelize3d, fetched 2026-10-27 (unofficial)
[S4] eternalscholar/Transforming-3D-LiDAR-point-cloud-data-to-2D-images-through-voxelization, https://github.com/eternalscholar/Transforming-3D-LiDAR-point-cloud-data-to-2D-images-through-voxelization, fetched 2026-10-27 (unofficial)
[S5] sgffoen/PointCloudVoxelization, https://github.com/sgffoen/PointCloudVoxelization, fetched 2026-10-27 (unofficial)
[S6] https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py#L305, https://github.com/google-research/google-research/blob/master/tf3d/utils/voxel_utils.py, fetched 2026-10-27 (unofficial)
