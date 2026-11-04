# Hard Voxelization

## Overview
Hard Voxelization converts a continuous 3D point cloud into a discrete, sparse voxel grid by assigning each point to a specific voxel cell based on its spatial coordinates and a fixed voxel size [S2, unofficial]. It differs from soft voxelization by using hard assignment (no interpolation) and from sparse voxel octrees by using a fixed-axis-aligned grid structure [S2, unofficial]. This method serves as the foundational step for producing volume data from raw point clouds, building on [Point Cloud Voxelization](1000-point-cloud-voxelization.md).

## Shared principles
All variants in this family share the core mechanism of mapping continuous point coordinates $(x, y, z)$ to discrete integer voxel indices $(i, j, k)$ using floor division relative to a bounding box origin and voxel dimensions.

Given a point $p = [x, y, z]$, a voxel size $v = [v_x, v_y, v_z]$, and a range $R = [x_{min}, y_{min}, z_{min}, x_{max}, y_{max}, z_{max}]$, the voxel coordinates are computed as:
$$
i = \left\lfloor \frac{x - x_{min}}{v_x} \right\rfloor, \quad
j = \left\lfloor \frac{y - y_{min}}{v_y} \right\rfloor, \quad
k = \left\lfloor \frac{z - z_{min}}{v_z} \right\rfloor
$$
[S1, S2, unofficial]

Points with coordinates outside the range $R$ are discarded or mapped to invalid indices (e.g., $-1$) [S1, S2, unofficial]. The output is typically a set of tensors: `voxels` (point features grouped by voxel), `coors` (voxel indices), and `num_points_per_voxel` (counts) [S1, S2, unofficial].

## Subtopics

### [Multiscale Hard Voxelization](3319-multiscale-hard-voxelization.md)
Multiscale Hard Voxelization voxelizes a point cloud into multiple spatial scales simultaneously, outputting voxel features, coordinates, and point counts for each scale. It differs from standard single-scale voxelization by iterating over multiple scales and storing results in separate memory blocks indexed by `scale_idx`, providing convenience for multi-scale feature extraction networks.

### [Hard Voxelization (Deterministic)](3321-hard-voxelization-deterministic.md)
Hard Voxelization (Deterministic) converts a 3D point cloud into a fixed-size voxel grid by assigning each point to a discrete voxel based on its coordinates and a specified voxel size and range. It handles duplicate points within the same voxel and limits the number of points per voxel and total voxels. Unlike the non-deterministic variant which uses atomic operations for speed at the cost of reproducibility, this method guarantees bit-identical output by using stable sorting and explicit ordering logic. It serves as the foundational step for producing volume data from raw point clouds, building on [Hard Voxelization](5345-hard-voxelization.md).

### [Point Cloud Voxelization Visualization](5628-point-cloud-voxelization-visualization.md)
A visualization pipeline that renders voxelized point clouds using the Point Cloud Library (PCL), supporting axis-aligned or custom bounding boxes with three distinct rendering modes. It differs from its siblings by providing immediate visual feedback of voxel occupancy and bounding constraints through specific geometric representations (points, solid cubes, or wireframes) rather than just generating the underlying data structure.

#### Dynamic Voxelization (Coordinate Only)
- **Definition**: Computes only the voxel coordinates for each point, without assigning points to a fixed voxel grid structure or handling duplicates/max points. Used when `max_points=-1` [S1, unofficial].
- **Steps**: Calculate grid dimensions: $grid_x = \text{round}((coors\_x_{max} - coors\_x_{min}) / voxel\_x)$, etc. Launch `dynamic_voxelize_kernel` to compute voxel coordinates for each point and store them in the `coors` tensor [S1, unofficial].
- **Parameters**: `NDim`: Number of dimensions for coordinates, default 3 [S1, unofficial].

## Background
Hard voxelization is widely used in 3D object detection for autonomous driving, particularly in frameworks like MMDetection3D and OpenPCDet, to convert LiDAR point clouds into a format suitable for 3D Convolutional Neural Networks (CNNs) [S1, S2, unofficial]. The deterministic variant was developed to address reproducibility issues in non-deterministic CUDA implementations where atomic operations could lead to race conditions and varying outputs across runs [S2, unofficial]. Multiscale variants were introduced to facilitate feature extraction at different resolutions without multiple passes [S1, unofficial].

## Sources
[S1] Susie0909/Multiscale_Voxelization_CUDA, https://github.com/Susie0909/Multiscale_Voxelization_CUDA, fetched 2026-10-27 (unofficial)
