# Index Computation and Mapping

Also called: Voxel Indexing, Grid Coordinate Mapping

## Overview
Index Computation and Mapping provides the fundamental mechanism for translating continuous 3D geometric coordinates into discrete linear array indices within a voxel grid. It solves the problem of spatial lookup in dense volume representations by converting geometric positions into memory offsets, enabling efficient access to voxel data. It sets itself apart from [Uniform Grid Adjacent Cell Indexing](2992-uniform-grid-adjacent-cell-indexing.md) by focusing on single-point primary lookup rather than neighborhood traversal, and from [Sparse Voxel Storage and Chunking](15595-sparse-voxel-storage-and-chunking.md) by operating on dense, regularly spaced grids rather than hierarchical sparse structures.

## Shared principles
All methods in this family rely on the linearization of 3D grid coordinates $(i, j, k)$ into a 1D integer index $I$. The general form is:
$$ I = i \cdot S_x + j \cdot S_y + k \cdot S_z $$
where $S_x, S_y, S_z$ are strides determined by the grid dimensions. The most common convention is row-major ordering where $X$ is the fastest-varying dimension (inner loop), $Y$ is middle, and $Z$ is slowest (outer loop), resulting in $S_x = N_y N_z$, $S_y = N_z$, and $S_z = 1$ for a grid of size $N_x \times N_y \times N_z$.

Coordinate transformation is a prerequisite step. Continuous world positions $\mathbf{p}$ are mapped to grid coordinates via:
1.  **Translation**: $\mathbf{p}_{rel} = \mathbf{p} - \mathbf{b}_{min}$, where $\mathbf{b}_{min}$ is the grid's minimum bound.
2.  **Scaling**: $\mathbf{c}_{float} = \mathbf{p}_{rel} / \mathbf{s}$, where $\mathbf{s}$ is the voxel size (or derived from bounds and divisions).
3.  **Discretization**: Integer indices are obtained via floor operation: $i = \lfloor c_{float,x} \rfloor$, etc.

Boundary handling is critical. Points outside the axis-aligned bounding box (AABB) must be clamped to the valid range $[0, N-1]$ to prevent out-of-bounds memory access, rather than raising errors or returning invalid indices, unless the specific application (like edge-based marching cubes) requires explicit boundary detection.

## Subtopics

### [Uniform Grid Adjacent Cell Indexing](2992-uniform-grid-adjacent-cell-indexing.md)
Computes the four linear indices of cells sharing a specific edge. Used for edge-based operations like Marching Cubes. Distinct by mapping an edge geometry to a set of neighbors rather than a single cell.

### [Uniform Grid Indexing](6941-uniform-grid-indexing.md)
Maps a continuous 3D position to a single linear index in a uniform grid. The standard approach for dense volume lookup with strict AABB constraints.

### [Voxel Index Computation](13870-voxel-index-computation.md)
Supports batched point-to-grid mapping for Gaussian Splatting and specialized 3-bit texel indexing for BC6H compression. Distinct by handling batched inputs and non-standard compression contexts.

## Background
Originating from standard computer graphics techniques for texture mapping and volume rendering, this family of methods is foundational to any rasterization or ray-marching pipeline that uses voxelized data. The linearization formula $I = i \cdot N_y N_z + j \cdot N_z + k$ is a standard convention in 3D array storage (e.g., in C/C++ and Python NumPy). The specific handling of boundaries and floating-point precision has evolved with the adoption of GPU-based rendering, where divergence-free branching and consistent clamping are preferred for performance.

## Sources
