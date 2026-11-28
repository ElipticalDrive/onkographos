# Gaussian to Voxel Conversion via Grid/Kernel
Also called: Gaussian Voxelization, Gaussian Grid Mapping

## Overview
This family converts continuous 3D Gaussian primitives into discrete voxel grid representations for efficient processing by 3D networks or differentiable renderers. It solves the problem of mapping continuous spatial distributions to a fixed-resolution grid while preserving geometric and photometric information. It differs from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) by using analytical Gaussian kernels rather than simple occupancy or nearest-neighbor assignment, and from [De-voxelization](8432-de-voxelization.md) by operating in the forward direction (continuous to discrete).

## Shared principles
The core mechanism involves mapping continuous Gaussian primitives $G_k$ (defined by mean $\mu_k$, covariance $\Sigma_k$, opacity $\alpha_k$, and features $F_k$) to a discrete voxel grid. Common principles include:
1.  **Spatial Mapping**: Determining which voxels a Gaussian influences, typically via bounding box checks or distance kernels.
2.  **Aggregation**: Combining contributions from multiple Gaussians into a single voxel cell. This often involves weighted sums based on opacity or kernel distance.
3.  **Discretization**: Converting continuous attributes (like position or color) into grid-aligned values, often requiring handling of floating-point precision and boundary conditions.
4.  **Sparsification**: Reducing the density of the voxel grid by culling empty or low-confidence voxels to manage memory and computational load.

## Subtopics

### [Gaussian Kernel Interpolation](17203-gaussian-kernel-interpolation.md)
Uses a Gaussian weight function based on distance to interpolate values from scattered points to voxel centers. It produces smoother results than Nearest Neighbor interpolation but is sensitive to non-uniform point distributions when using a fixed $k$ [17203].

### [Gaussian Preprocessing and Voxelization](19076-gaussian-preprocessing-and-voxelization.md)
Computes voxel-space bounding boxes and conic matrices (inverse covariance) for each Gaussian to determine overlap. It focuses on pre-rasterization metadata generation and analytical bounding rather than kernel density estimation [19076].

### [3DGR-CT Gaussian Voxelization](20578-3dgr-ct-gaussian-voxelization.md)
A CUDA-based method for sparse-view CT reconstruction that explicitly constructs a volumetric density field. It uses a work-stealing kernel with shared memory caching and atomic operations to handle variable-sized influence regions and ensure differentiability [20578].

## Background
This family is foundational for 3D Gaussian Splatting variants that require volumetric representations, such as those used in CT reconstruction or neural rendering pipelines that operate on voxel grids rather than point clouds. The transition from point-based to voxel-based representations allows for the application of standard 3D CNNs or differentiable volume renderers to Gaussian primitives.

