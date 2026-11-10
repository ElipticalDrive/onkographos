# Hard Voxelization
## Overview
Hard Voxelization discretizes a continuous 3D point cloud into a fixed-size 3D grid of cubic voxels by binning points based on their spatial coordinates and aggregating associated attribute values within each voxel [S3, unofficial]. It solves the problem of converting unstructured point data into a structured tensor format suitable for 3D convolutional neural networks [S2, unofficial]. It differs from its siblings by enforcing a strict grid structure with fixed voxel sizes, whereas [Gaussian Voxelization](8884-gaussian-voxelization.md) uses soft assignments and [Octree Construction for Point Clouds](1070-octree-construction-for-point-clouds.md) uses hierarchical variable-resolution structures.
## Shared principles
The core mechanism maps continuous point coordinates $(x, y, z)$ to discrete integer voxel indices $(i, j, k)$ using floor division relative to a bounding box origin and voxel dimensions [S1, S2, S3, unofficial].

For a point $p$ with coordinates $(x, y, z)$, a voxel size $v = [v_x, v_y, v_z]$, and a bounding box range $R = [x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$, the voxel coordinates are computed as:
$$
i = \left\lfloor \frac{x - x_{min}}{v_x} \right\rfloor, \quad
j = \left\lfloor \frac{y - y_{min}}{v_y} \right\rfloor, \quad
k = \left\lfloor \frac{z - z_{min}}{v_z} \right\rfloor
$$
[S1, S2, S3, unofficial]

Points with coordinates outside the range $R$ are mapped to invalid indices (e.g., $-1$) and are typically discarded or filtered out [S1, S2, unofficial]. The grid dimensions are calculated as:
$$
\text{dim}_x = \max(1, \lceil \frac{x_{max} - x_{min}}{v_x} \rceil), \quad
\text{dim}_y = \max(1, \lceil \frac{y_{max} - y_{min}}{v_y} \rceil), \quad
\text{dim}_z = \max(1, \lceil \frac{z_{max} - z_{min}}{v_z} \rceil)
$$
[S1, unofficial]

The output consists of three tensors: `voxels` containing point features grouped by voxel, `coors` containing the integer voxel indices, and `num_points_per_voxel` storing the count of points in each voxel [S2, unofficial].
## Subtopics

### [Multiscale Hard Voxelization](3319-multiscale-hard-voxelization.md)
Multiscale Hard Voxelization is a GPU-accelerated method that simultaneously voxelizes a LiDAR point cloud into multiple spatial resolutions (scales) in a single pass. It extends standard hard voxelization by iterating over a list of voxel sizes and point limits, outputting separate voxel feature tensors, coordinates, and point counts for each scale. It differs from single-scale voxelization by managing multiple memory blocks indexed by scale, offering a 10% speed improvement over ordinary single-scale CUDA operators on NVIDIA RTX 3090 hardware.

### [Hard Voxelization (Deterministic)](3321-hard-voxelization-deterministic.md)
Hard Voxelization (Deterministic) discretizes a continuous 3D point cloud into a fixed-size 3D grid of cubic voxels by binning points based on their spatial coordinates and aggregating associated attribute values within each voxel. It outputs a single representative point per occupied voxel calculated as the centroid of the aggregated attributes. Unlike its sibling [Multiscale Hard Voxelization](3319-multiscale-hard-voxelization.md), which handles multiple resolutions, this method operates on a single fixed scale. It differs from the non-deterministic variant by guaranteeing reproducible output through explicit sorting and unique-coordinate identification, ensuring that the order of points within a voxel is preserved, whereas non-deterministic versions rely on atomic operations that may yield varying results across runs.

### [Point Cloud Voxelization Visualization](5628-point-cloud-voxelization-visualization.md)
Point Cloud Voxelization Visualization is a rendering pipeline that visualizes the occupancy of a point cloud after it has been downsampled into a 3D voxel grid using the Point Cloud Library (PCL). It solves the problem of visually verifying voxelization parameters and bounding box constraints by offering three distinct rendering modes: raw point cloud coloring, solid voxel cubes, and wireframe voxel cubes. It differs from its siblings by focusing specifically on the *visualization* of the filtered output rather than just the aggregation logic, and it relies on PCL's `VoxelGrid` filter for the downsampling step.

### Dynamic Voxelization (Coordinate Only)
- **Definition**: Computes only the voxel coordinates for each point, without assigning points to a fixed voxel grid structure or handling duplicates/max points. Used when `max_points=-1` [S1, unofficial].
- **Steps**: Calculate grid dimensions: `grid_x = round((coors_x_max - coors_x_min) / voxel_x)`, etc. Launch `dynamic_voxelize_kernel` to compute voxel coordinates for each point and store them in the `coors` tensor [S1, unofficial].
- **Parameters**: `NDim`: Number of dimensions for coordinates, default 3 [S1, unofficial].

## Background
Hard voxelization is a fundamental preprocessing step in 3D deep learning, particularly for autonomous driving perception tasks using LiDAR data [S2, unofficial]. The deterministic variant and multiscale extensions are heavily utilized in frameworks like MMDetection3D and OpenPCDet, which rely on CUDA implementations for efficient GPU processing [S1, unofficial]. The technique originated from the need to convert sparse, unstructured point clouds into dense, structured grids compatible with 3D Convolutional Neural Networks (3D CNNs) [S2, unofficial].

## Sources
[S1] Susie0909/Multiscale_Voxelization_CUDA, https://github.com/Susie0909/Multiscale_Voxelization_CUDA, fetched 2026-10-27 (unofficial)
