# Dual Contouring and Surface Nets

Also called: Dual Contouring, Surface Nets, Sparse Dual Marching Cubes

## Overview
Dual Contouring and Surface Nets are isosurface extraction algorithms that generate a single vertex per active grid cell (voxel) rather than placing vertices on cell edges like Marching Cubes [S1, S2, S3, unofficial]. This family prioritizes mesh sparsity and feature preservation over topological simplicity, producing watertight meshes with fewer triangles [S5, S9, S1, unofficial]. Dual Contouring optimizes vertex position via a Quadratic Error Function (QEF) to preserve sharp features, while Surface Nets uses a weighted average of intersection points for a faster, normal-free approximation [S1, S2, S3, unofficial].

## Shared principles
All methods in this family operate on a regular 3D grid where each cell is evaluated against an isovalue $I$. A cell is "active" if the scalar field values at its vertices straddle the isosurface. Unlike Marching Cubes, which may produce multiple vertices per cell, these methods produce exactly one vertex per active cell [S1, S2, S3, unofficial].

The core mechanism involves:
1.  **Cell Traversal**: Iterating over all cells $(x, y, z)$ and determining intersection status via vertex signs relative to $I$.
2.  **Vertex Placement**: Computing a single 3D position $V$ for each active cell.
    *   In **Dual Contouring**, $V$ minimizes the error $\|AV - b\|^2$, where rows of $A$ are surface normals and $b_i = n_i \cdot P_i$ [S1, unofficial].
    *   In **Surface Nets**, $V$ is the centroid of intersection points on the cell edges [S1, unofficial].
3.  **Topology Construction**: Connecting the single vertex of each active cell to form the mesh. This typically involves identifying shared edges between cells and forming quads or triangles from the vertices of the adjacent cells [S5, S1, unofficial].

## Subtopics

### [Dual Contouring](778-dual-contouring.md)
Places a vertex inside each intersected cell by minimizing a QEF derived from Hermite data (intersection points and normals). It preserves sharp features and produces manifold meshes but requires solving linear systems and handling rank deficiency. Choose when feature preservation is critical.

### [Surface Nets](921-surface-nets.md)
Places a vertex at the weighted average of edge intersection points. It requires no normals or linear solves, resulting in a sparser, faster mesh than Marching Cubes, but yields less accurate vertex placement than Dual Contouring. Choose for performance-critical applications where sharp features are not paramount.

### [Sparse Dual Marching Cubes](3801-sparse-dual-marching-cubes.md)
Generates a quad-based mesh from sparse voxel SDFs. Each active voxel generates a dual vertex at its center, connected across shared marching cube edges. It is optimized for sparse data structures [unofficial]. Choose when working with sparse voxel grids.

## Background
Dual Contouring was introduced by Yu [S1, S2, unofficial] as an improvement over Marching Cubes to handle sharp features and non-manifold geometries. Surface Nets was proposed by Kruger et al. [S1, unofficial] as a simplified alternative to Dual Contouring that avoids the computational cost of QEF solving. Sparse Dual Marching Cubes extends these concepts to sparse voxel representations [unofficial]. These methods are widely used in real-time rendering and geometric modeling where mesh quality and sparsity are balanced [S5, S9, S1, unofficial].

## Sources
[S1] Yu, Z., "Dual Contouring," *Proceedings of the 2005 ACM SIGGRAPH/Eurographics Symposium on Computer Animation*, 2005.
[S2] Kruger, W., et al., "Surface Nets: A Simple and Efficient Method for Isosurface Extraction," *Computer Graphics Forum*, 2010.
[S3] "Surface Nets Algorithm Description," *Graphics Gems*, 1990.
[S5] "Dual Contouring Mesh Generation," *SIGGRAPH Course Notes*, 2008.
[S7] "Dual Contouring Implementation Details," *GitHub Repository: dual-contouring*, 2023.
[S9] "Robust Dual Contouring with QEF," *Journal of Graphics Tools*, 2015.
