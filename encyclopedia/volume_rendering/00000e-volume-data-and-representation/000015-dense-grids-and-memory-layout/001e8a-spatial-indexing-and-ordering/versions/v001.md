# Spatial Indexing and Ordering

Also called: Spatial Indexing and Compound Ordering

## Overview
Spatial Indexing and Ordering provides the mechanisms to map multi-dimensional voxel coordinates to linear memory indices and to organize data in memory to maximize spatial locality for cache-coherent access. It solves the problem of inefficient memory access patterns in volume rendering by ensuring that spatially adjacent voxels are stored in adjacent memory locations, thereby reducing cache misses during ray marching and data loading. It sets itself apart from hierarchical indexing methods (like [Morton Code Generation for Octrees](4673-morton-code-generation-for-octrees.md)) by focusing on flat linearization and sorting strategies rather than tree-based culling, and from pure storage formats by defining the algorithmic mapping between logical space and physical memory.

## Shared principles
All methods in this family share the goal of preserving spatial locality in linear memory. The core mechanism involves transforming $N$-dimensional coordinates $(x, y, z)$ into a 1D index $I$ such that points close in Euclidean distance have close indices $I$. This is achieved through:
1.  **Linearization:** Mapping 3D grid coordinates to a 1D array using arithmetic strides (row-major) or bit-interleaving (Morton/Z-order).
2.  **Locality Preservation:** Ensuring that the mapping function minimizes the distance between indices of spatially neighboring voxels.
3.  **Coordinate Transformation:** Converting world-space ray positions into grid-local indices using transformation matrices or scaling factors.

The mathematical foundation for simple linearization is the row-major formula:
$$ I = x \cdot (Y \cdot Z) + y \cdot Z + z $$
where $X, Y, Z$ are the grid dimensions. For space-filling curves, the mapping is non-linear but preserves locality better than simple strides.

## Subtopics

### [Grid Density Data Structure](998-grid-density-data-structure.md)
A dense, axis-aligned 3D grid storing scalar density values in a contiguous 1D array. It solves the problem of representing continuous volumetric media as a discrete field for efficient ray marching. Unlike sparse structures, it offers $O(1)$ sampling speed by eliminating pointer chasing, at the cost of $O(N^3)$ memory usage. It builds on [Spatial Indexing and Ordering](1797-spatial-indexing-and-ordering.md) by providing the underlying memory layout. See [Grid Density Data Structure](998-grid-density-data-structure.md).

### [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md)
Converts N-dimensional integer coordinates into a single 1D integer (Morton code) by interleaving bits, and reverses this process. It preserves spatial locality, mapping nearby points to nearby indices, which accelerates cache-coherent access. It differs from hierarchical methods by operating on flat Cartesian coordinates rather than tree paths. See [Morton Code Encoding/Decoding](1700-morton-code-encoding-decoding.md).

### [Voxel Grid Indexing](1719-voxel-grid-indexing.md)
Maps 3D voxel coordinates $(i, j, k)$ to a linear 1D memory index using arithmetic strides, enabling direct access to dense volume data. It provides deterministic $O(1)$ access but assumes a fully dense grid, resulting in $O(N^3)$ storage. It differs from [Grid Density Data Structure](998-grid-density-data-structure.md) by focusing on the coordinate-to-index arithmetic rather than density-based culling. See [Voxel Grid Indexing](1719-voxel-grid-indexing.md).

### Z-Order and Hilbert Encoding/Decoding
CPU and CUDA implementations for Z-order and Hilbert space-filling curve encoding and decoding, used for serialization and sorting [S1, unofficial].

### Spatial Indexing and Compound Ordering
Sorts elements first by discrete non-spatial coordinates (time, channel), and within each slice, along a space-filling curve over spatial coordinates [S2, unofficial]. Precomputes chunk-level AABBs for fast view-dependent fetching [S2, unofficial]. Hilbert is the default for serialization due to superior locality; Morton is used for in-memory reordering due to lower computational cost [S2, unofficial].

## Background
The concept of linearizing multi-dimensional data for cache efficiency is fundamental to computer graphics. Morton codes (Z-order) were introduced by G. M. Morton in 1966 [S7, unofficial]. Hilbert curves provide even better locality but are more complex to compute [S2, unofficial]. Compound ordering for scientific data visualization, combining non-spatial and spatial sorting, was described in the context of the Luxar viewer for microscopy data [S2, unofficial].

## Sources
[S1] quantaji/o-voxel-gpu, https://github.com/quantaji/o-voxel-gpu, fetched 2026-10-29 (unofficial)
[S2] Luxar: Gaussian splatting for microscopy and scalable interactive web visualisation of multidimensional scientific data, https://doi.org/10.64898/2026.09.22.753676, fetched 2026-10-26 (unofficial)
