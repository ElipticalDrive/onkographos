# Hard Voxelization

Also called: Point Cloud Hard Voxelization, Fixed-Size Voxelization

## Overview
Hard Voxelization discretizes a continuous 3D point cloud into a fixed-size 3D grid of cubic voxels by binning points based on their spatial coordinates and aggregating associated attribute values within each voxel [S3, unofficial]. It solves the problem of converting unstructured point data into a regular tensor format suitable for convolutional neural networks or efficient spatial indexing. It differs from [Gaussian Voxelization](8884-gaussian-voxelization.md) by using binary occupancy or simple aggregation rather than density estimation, and from [Ray Model Voxelization](4688-ray-model-voxelization.md) by relying on direct coordinate mapping rather than ray casting.

## Shared principles
The core mechanism maps continuous coordinates $(x, y, z)$ to discrete integer voxel indices $(i, j, k)$ using floor division relative to a bounding box origin and voxel dimensions [S1, unofficial].

1.  **Bounding Box Definition**: The spatial extent is defined by $[x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$ [S3, unofficial].
2.  **Grid Dimension Calculation**: The number of voxels along each axis is calculated. Sources differ on rounding:
    *   Multiscale variant uses rounding: $grid_x = \text{round}\left(\frac{x_{max} - x_{min}}{v_x}\right)$ [S1, unofficial].
    *   Deterministic variant uses ceiling: $\text{dim}_x = \max(1, \lceil \frac{x_{max} - x_{min}}{v_x} \rceil)$ [S1, unofficial].
3.  **Coordinate Mapping**: For each point $p$ with coordinates $(x, y, z)$, the voxel index is computed as:
    $$
    i = \left\lfloor \frac{x - x_{min}}{v_x} \right\rfloor, \quad
    j = \left\lfloor \frac{y - y_{min}}{v_y} \right\rfloor, \quad
    k = \left\lfloor \frac{z - z_{min}}{v_z} \right\rfloor
    $$
    [S1, S2, S3, unofficial]
4.  **Boundary Handling**: Points with indices outside $[0, grid_x)$, $[0, grid_y)$, or $[0, grid_z)$ are discarded or marked as invalid (e.g., index $-1$) [S1, S2, unofficial].
5.  **Aggregation**: Points falling into the same voxel are aggregated. Common strategies include storing all points up to a limit, or computing the centroid (average attributes) [S1, S4, unofficial].
6.  **Output Structure**: The result is typically a tensor of shape $[M, \text{max\_points}, D]$ for features, $[M, 3]$ for voxel coordinates, and $[M]$ for point counts, where $M$ is the number of occupied voxels [S2, unofficial].

## Subtopics

### [Multiscale Hard Voxelization](3319-multiscale-hard-voxelization.md)
Multiscale Hard Voxelization is a GPU-accelerated method that simultaneously voxelizes a LiDAR point cloud into multiple spatial resolutions (scales) in a single pass. It extends standard hard voxelization by iterating over a list of voxel sizes and point limits, outputting separate voxel feature tensors, coordinates, and point counts for each scale. It differs from single-scale voxelization by managing multiple memory blocks indexed by scale, offering a 10% speed improvement over ordinary single-scale CUDA operators on NVIDIA RTX 3090 hardware.

### [Hard Voxelization (Deterministic)](3321-hard-voxelization-deterministic.md)
Hard Voxelization (Deterministic) discretizes a continuous 3D point cloud into a fixed-size 3D grid of cubic voxels by binning points based on their spatial coordinates and aggregating associated attribute values within each voxel. It outputs a single representative point per occupied voxel calculated as the centroid of the aggregated attributes. Unlike its sibling [Multiscale Hard Voxelization](3319-multiscale-hard-voxelization.md), which handles multiple resolutions, this method operates on a single fixed scale. It differs from the non-deterministic variant by guaranteeing reproducible output through explicit sorting and unique-coordinate identification, ensuring that the order of points within a voxel is preserved, whereas non-deterministic versions rely on atomic operations that may yield varying results across runs.

### [Point Cloud Voxelization Visualization](5628-point-cloud-voxelization-visualization.md)
This method visualizes the output of hard voxelization by rendering the input point cloud and/or the resulting voxel grid structure using the Point Cloud Library (PCL). It solves the problem of verifying voxelization correctness and inspecting spatial occupancy by offering three distinct rendering modes: point cloud coloring, solid cubes, and wireframe cubes. It differs from standard hard voxelization by focusing on the visual representation of the grid cells and their centers rather than just producing a tensor for downstream neural network processing.

### Dynamic Voxelization (Coordinate Only)
- [S1, unofficial] Computes only the voxel coordinates for each point, without assigning points to a fixed voxel grid structure or handling duplicates/max points. Used when max_points=-1.
- [S1, unofficial] Calculate grid dimensions: grid_x = round((coors_x_max - coors_x_min) / voxel_x), etc.
- [S1, unofficial] Launch dynamic_voxelize_kernel to compute voxel coordinates for each point and store them in the coors tensor.
- [S1, unofficial] NDim: Number of dimensions for coordinates, default 3.

## Background
Hard voxelization is a fundamental preprocessing step in 3D deep learning, particularly for LiDAR-based autonomous driving perception. It was popularized by frameworks like MMDetection3D and PointPillars, which require regular grid inputs for 3D CNNs or 2D projection networks [S1, unofficial]. The deterministic variant addresses the non-determinism inherent in GPU atomic operations, which is critical for debugging and reproducibility in training pipelines [S3, unofficial].

## Sources
[S1] Susie0909/Multiscale_Voxelization_CUDA, https://github.com/Susie0909/Multiscale_Voxelization_CUDA, fetched 2026-10-27 (unofficial)
