# Hierarchical Adaptive Voxelization

Also called: Adaptive Voxel Octree, Sparse Voxel Octree Construction for Unstructured Grids

## Overview
Hierarchical Adaptive Voxelization converts complex geometric inputs, such as unstructured grids or parametric surfaces, into a sparse, multi-resolution voxel structure (typically an octree) to enable efficient rendering or reconstruction. It solves the problem of memory inefficiency and sampling artifacts inherent in uniform grids by allocating high resolution only where geometric or data complexity demands it. Unlike fixed-resolution methods, it dynamically partitions space based on error metrics or geometric features, differing from sibling methods like [Solid Voxelization via Octree](6830-solid-voxelization-via-octree.md) by focusing on adaptive data density rather than binary occupancy, and from [Neural Sparse Voxel Fields (NSVF)](7144-neural-sparse-voxel-fields-nsvf.md) by using explicit geometric primitives rather than neural implicit representations.

## Shared principles
The family shares the core mechanism of recursive spatial partitioning (typically octrees) driven by local complexity metrics.
1.  **Adaptive Partitioning:** Space is divided into nodes. A node is subdivided if a local error metric (e.g., scalar field variance, curvature, or distance to surface) exceeds a threshold.
2.  **Hierarchical Storage:** Data is stored in bricks or tiles. Leaf nodes store the actual data (scalar values, normals, or geometric primitives), while internal nodes store metadata or aggregated filters.
3.  **Boundary Handling:** To prevent artifacts at the interface of different resolutions, overlapping regions (borders) are managed via stencil textures, padding, or specific filtering kernels that ensure continuity across level-of-detail (LoD) boundaries.

## Subtopics

### [Adaptive Voxelization of Unstructured Grids](3246-adaptive-voxelization-of-unstructured-grids.md)
Converts large, out-of-core unstructured grids (tetrahedral or hexahedral meshes) into a hierarchical 3D texture (voxel octree) for interactive volume rendering. It differs from multi-triangulation approaches by relying on structured grid hierarchies and 3D texture mapping acceleration. It uses a top-down recursive construction where tetrahedra are distributed to child nodes; recursion stops when scalar-field homogeneity or tetrahedra size thresholds are met. It employs a bottom-up filtering stage to merge children and compute gradients, using stencil textures to prevent striping artifacts at data boundaries.

### [Sparse Voxelization with Hierarchical LoD](7760-sparse-voxelization-with-hierarchical-lod.md)
A pipeline for high-frequency sparse geometric objects (fibers, hair, thin planes) that processes volumetric data into multiple Levels of Detail (LoD). It differs from general-purpose voxelizers by explicitly handling microgeometry via subsampling and encoding directional distributions into SGGX functions, rather than relying on simple occupancy. It separates geometric/directional data from material properties, using GPU kernels to filter empty blocks and aggregate samples into voxel nodes containing SGGX parameters.

### [Pyramid Pivot Voxelization](14366-pyramid-pivot-voxelization.md)
An adaptive strategy that exploits spatial non-uniformity by allocating finer pivot tokens to geometrically complex regions while keeping smooth regions coarse. It builds on [Voxelization](12-voxelization.md) by introducing a block-wise pyramid sampling map that assigns variable resolutions to macro-blocks based on a complexity score (combining curvature, gradient, and normal deviation). Unlike fixed-resolution methods, it reserves fine-scale sampling for high-frequency details, improving Pareto efficiency over SDF-based and Dual Contouring baselines. It lifts tokens to a maximum resolution for unified Sparse Marching Cubes reconstruction.

### Adaptive Binary Subdivision for Parametric Surface Voxelization
An adaptive technique to determine the sampling rate for voxelizing parametric surfaces by hierarchically subdividing the parameter interval $Z$, replacing uniform sampling to avoid oversampling in smooth regions and undersampling in complex regions [S1]. For each sample point, the distance to the surface is computed; if not easily computable, $g(p)=0$ is used for implicit solids [S1]. The current patch is subdivided into two smaller patches (binary subdivision) instead of four (quadtree) [S1]. Subdivision continues until the patch size $SP$ reaches a predefined value [S1]. For each leaf of the binary tree, a single sample is taken at its center [S1]. Quadtree subdivision causes large differences between patch sizes, leading to unsatisfactory results [S1]. For a sphere with diameter 51.2 and a Bardett filter with radius 1.6, the optimal number of samples was about $3.5 \cdot 10^5$, corresponding to a step between samples of about 0.5 voxel units [S1]. Improvement in surface quality is negligible for lookup table arrays with size above $40^3$ [S1].

## Background
The concept of adaptive voxelization for unstructured grids was introduced by Geresim and Spike in the context of interactive volume rendering on standard PC hardware . The method was developed to handle large datasets that exceed main memory, utilizing a CPU-based preprocessing stage to build the octree and a GPU-based rendering stage . The adaptive binary subdivision for parametric surfaces was explored in early filtering-based voxelization research to optimize sampling density for implicit and parametric objects [S1].

## Sources
[S1] Object voxelization by filtering, https://doi.org/10.1145/288126.288178, fetched 2026-11-03
