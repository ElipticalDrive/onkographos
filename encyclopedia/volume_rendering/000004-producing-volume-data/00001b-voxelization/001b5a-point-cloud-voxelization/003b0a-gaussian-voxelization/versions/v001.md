# Gaussian Voxelization
Also called: Gaussian Splat Voxelization, 3D Gaussian Voxel Grid Conversion

## Overview
Gaussian Voxelization converts continuous 3D Gaussian distributions (splat representations) into discrete voxel grids or sparse octrees, enabling efficient processing by 3D neural networks, collision detection, and navigation systems [S1, unofficial]. It distinguishes itself from hard voxelization by sampling voxel locations based on the geometric extent and density of each Gaussian rather than simply rasterizing point centers, allowing for denser feature embedding and smoother representations [S1, unofficial].

## Shared principles
All methods in this family operate on a set of 3D Gaussians, where each Gaussian $k$ is defined by a center position $c_k$, a scale (or variance $\sigma_k$), and potentially a covariance matrix $\Sigma_k$. The core mechanism involves mapping these continuous probability distributions to discrete grid cells. This requires determining which voxels are influenced by a Gaussian, typically by evaluating the Gaussian's density function against the voxel boundaries or by sampling points within the Gaussian's effective support volume. The output is a sparse or dense set of voxels with associated features (e.g., opacity, color, density) aggregated from the contributing Gaussians.

## Subtopics

### [Point Cloud to Gaussian Converter](1788-point-cloud-to-gaussian-converter.md)
Transforms a discrete point cloud into isotropic Gaussian ellipsoids. It differs from other methods by generating the Gaussians themselves from point proximity, using a coverage rate $\zeta$ and neighbor count $m$ to determine variance. Choose this when starting from raw points rather than pre-existing splats.

### [Voxel Format Generation](6914-voxel-format-generation.md)
Converts 3D Gaussian Splats into a sparse voxel octree optimized for navigation and collision. It integrates scene topology analysis (interior/exterior classification) and agent-based navigation carving directly into the pipeline. Choose this for robotics or simulation applications requiring navigable meshes.

### [3D Gaussian Voxelization](7020-3d-gaussian-voxelization.md)
Converts 3D Gaussians into unique voxels and features for 3D neural networks. It samples voxel locations based on the geometric extent of each Gaussian using configurable shapes (volume, tri-plane, etc.). Choose this for deep learning pipelines requiring structured grid inputs from Gaussian representations.

## Background
The concept of voxelizing Gaussian splats emerged with the rise of 3D Gaussian Splatting (3DGS) for novel view synthesis [S1, unofficial]. As 3DGS moved from rendering to downstream tasks like reconstruction, navigation, and neural processing, methods were developed to bridge the gap between the unstructured, continuous Gaussian representation and the structured, discrete nature of voxel grids. This family of techniques addresses the need for efficient spatial indexing and feature aggregation in these hybrid representations.

## Sources
