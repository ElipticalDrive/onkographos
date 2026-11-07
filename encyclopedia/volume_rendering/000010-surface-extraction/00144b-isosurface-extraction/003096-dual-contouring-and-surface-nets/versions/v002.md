# Dual Contouring and Surface Nets

Also called: Dual Contouring, Surface Nets

## Overview
Dual Contouring and Surface Nets are isosurface extraction algorithms that generate a single vertex per active grid cell (voxel) rather than placing vertices on cell edges like Marching Cubes [G]. This family prioritizes mesh sparsity and feature preservation over topological simplicity, producing watertight meshes with fewer triangles [G]. Dual Contouring optimizes vertex position via a Quadratic Error Function (QEF) to preserve sharp features [778-dual-contouring.md], while Surface Nets uses a weighted average of intersection points for a faster, normal-free approximation [921-surface-nets.md].

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid, where each cell contains scalar values or occupancy states at its vertices or centers [G]. The core mechanism involves traversing the grid cells and evaluating the data to determine which parts of the domain constitute the surface [G]. A cell is "active" if the scalar field values at its vertices straddle the isosurface [G]. Unlike Marching Cubes, which may produce multiple vertices per cell, these methods produce exactly one vertex per active cell [778-dual-contouring.md][921-surface-nets.md].

The core mechanism involves:
1. **Cell Traversal**: Iterating over all cells $(x, y, z)$ and determining intersection status via vertex signs relative to $I$ [G].
2. **Vertex Placement**: Computing a single 3D position $V$ for each active cell.
    * In **Dual Contouring**, $V$ minimizes the error $\|AV - b\|^2$, where rows of $A$ are surface normals and $b_i = n_i \cdot P_i$ [778-dual-contouring.md].
    * In **Surface Nets**, $V$ is the centroid of intersection points on the cell edges [921-surface-nets.md].
3. **Topology Construction**: Connecting the single vertex of each active cell to form the mesh. This typically involves identifying shared edges between cells and forming quads or triangles from the vertices of the adjacent cells [G].

## Subtopics

### [Dual Contouring](778-dual-contouring.md)
Places a vertex inside each intersected cell by minimizing a QEF derived from Hermite data (intersection points and normals). It preserves sharp features and produces manifold meshes but requires solving linear systems and handling rank deficiency. Choose when feature preservation is critical.

### [Surface Nets](921-surface-nets.md)
Places a vertex at the weighted average of edge intersection points. It requires no normals or linear solves, resulting in a sparser, faster mesh than Marching Cubes, but yields less accurate vertex placement than Dual Contouring. Choose for performance-critical applications where sharp features are not paramount.

### [Sparse Dual Marching Cubes](3801-sparse-dual-marching-cubes.md)
Generates a quad-based mesh from sparse voxel SDFs. Each active voxel generates a dual vertex at its center, connected across shared marching cube edges. It is optimized for sparse data structures. Choose when working with sparse voxel grids.

## Background
Dual Contouring was introduced as an improvement over Marching Cubes to handle sharp features and non-manifold geometries [778-dual-contouring.md]. Surface Nets was proposed as a simplified alternative to Dual Contouring that avoids the computational cost of QEF solving [921-surface-nets.md]. Sparse Dual Marching Cubes extends these concepts to sparse voxel representations [3801-sparse-dual-marching-cubes.md].

## Sources
