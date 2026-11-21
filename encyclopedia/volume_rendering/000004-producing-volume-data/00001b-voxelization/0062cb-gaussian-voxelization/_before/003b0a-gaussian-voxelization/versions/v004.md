# Gaussian Voxelization

Also called: Gaussian-to-Voxel Conversion, 3DGS Voxelization, Splat-to-Voxel

## Overview
Gaussian Voxelization is the process of converting a continuous representation of 3D Gaussian primitives into a discrete voxel grid or sparse voxel structure. It solves the problem of bridging point-based Gaussian representations (such as those from 3D Gaussian Splatting) with grid-based neural processing, collision detection, or visualization engines that require structured volumetric data [S1, unofficial]. Unlike sibling methods that may focus on density estimation or direct splatting, this family explicitly constructs a spatial proxy that aggregates and distributes splat information, often serving as a preprocessing step for Sparse Voxel Transformers or physics simulations [S1, unofficial].

## Shared principles
All methods in this family share the fundamental mechanism of mapping continuous Gaussian distributions to discrete grid cells. The core operation involves identifying the spatial support of each Gaussian primitive and depositing its attributes (opacity, color, features) into the corresponding voxels. This process typically requires:
1.  **Spatial Mapping**: Determining which voxels intersect or are nearest to a Gaussian's mean position.
2.  **Weighting**: Computing the contribution of each Gaussian to a voxel, often based on the Gaussian density function or a distance kernel.
3.  **Aggregation**: Combining contributions from multiple Gaussians into a single voxel attribute, usually via weighted sums or averages.

The specific implementation details (e.g., isotropic vs. anisotropic Gaussians, sparse vs. dense grids, handling of spherical harmonics) vary by child method.

## Subtopics

### [Point Cloud to Gaussian Converter](1788-point-cloud-to-gaussian-converter.md)
Transforms a discrete point cloud into isotropic Gaussian ellipsoids by deriving scale from nearest-neighbor distances. Choose this when starting from unstructured points rather than pre-existing Gaussians. Differs from 3D Gaussian Voxelization by focusing on primitive generation.

### [Voxel Format Generation](6914-voxel-format-generation.md)
Converts Gaussians into a sparse voxel octree optimized for navigation and collision, including scene sealing and carving. Choose this for generating watertight, navigable meshes for physics engines. Differs by focusing on topological integrity and auxiliary mesh generation.

### [3D Gaussian Voxelization](7020-3d-gaussian-voxelization.md)
Converts Gaussians into unique voxels with an inverse mapping to preserve native structure for 3D convolutions. Choose this for neural network feature prediction. Differs by explicitly constructing a sparse voxel representation with inverse mapping.

### [Splat-to-Voxel Transfer](9930-splat-to-voxel-transfer.md)
Deposits splat attributes onto a structured grid using a distance kernel and supports coarse-to-fine hierarchy and sparsification. Choose this for preprocessing before a Sparse Voxel Transformer. Differs by handling continuous-to-discrete mapping with specific sparsification strategies.

### [3DGS Voxelizer](12530-3dgs-voxelizer.md)
Converts 3DGS scans into colored voxel blocks as OpenUSD `PointInstancer` data. Choose this for visualizing and enabling physics in NVIDIA Omniverse. Differs by targeting instanced cube primitives for real-time rendering and simulation.

## Background
The rise of 3D Gaussian Splatting (3DGS) [S1, unofficial] created a need to integrate these continuous, point-based representations into traditional voxel-based pipelines used in robotics, simulation, and neural rendering. Early methods focused on simple density estimation, but recent developments emphasize structured voxel grids that preserve geometric and photometric information for downstream tasks like collision detection and 3D convolution [S1, unofficial]. The field has diverged into specialized tools for specific engines (e.g., Omniverse) and general-purpose neural processing backends.

## Sources
[S1] 3D Gaussian Voxelization and Splat-to-Voxel Transfer documentation, unofficial notes, 2024.
