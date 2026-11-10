# Grid Traversal and Data Structure Adaptation

Also called: Adaptive Isosurface Extraction, Hierarchical Grid Traversal

## Overview
This family adapts standard isosurface extraction (typically Marching Cubes) to non-uniform or sparse data structures, optimizing memory usage and computational cost by focusing resolution only where the surface exists. It differs from uniform grid methods by dynamically determining which cells to evaluate based on spatial hierarchy or sparsity, and from crack-free AMR methods by relying on specific traversal or stitching logic to handle discontinuities between varying resolution levels [S1].

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid (Cartesian, octree, or tetrahedral) where each cell contains scalar values or occupancy states at its vertices or centers. The core mechanism involves traversing the grid cells and evaluating the data to determine which parts of the domain constitute the surface [G].

1.  **Domain Traversal**: The algorithm iterates over the volume data structure [G]. For regular grids, this often involves checking each cell or voxel [G].
2.  **Surface Determination**: A criterion is applied to identify surface locations. In isosurface methods, this involves comparing scalar values against a threshold (isovalue) to detect sign changes or equality [30-isosurface-extraction.md]. In blocky meshing, this involves checking occupancy states or material IDs to identify exposed boundaries [31-blocky-meshing.md].
3.  **Primitive Generation**: Geometric primitives (vertices, edges, faces) are generated based on the determined surface locations [G].
4.  **Mesh Coherence**: Adjacent primitives are merged or connected to form a coherent mesh, often requiring vertex merging to avoid duplicates and ensure watertightness [30-isosurface-extraction.md].

## Subtopics

### [Marching Cubes with Octree](1259-marching-cubes-with-octree.md)
Extracts a manifold triangle mesh from an implicit scalar field defined over an adaptive octree structure, replacing the uniform grid traversal of standard Marching Cubes with a hierarchical domain traversal. It sets itself apart from [Sparse Marching Cubes](3825-sparse-marching-cubes.md) by operating on a pointerless octree structure that adapts resolution based on a split predicate, and from [Hierarchical Marching Cubes Enumeration](5223-hierarchical-marching-cubes-enumeration.md) by relying on a specific domain object (`MC_Domain`) to handle the geometric mapping between octree nodes and cube vertices. Choose this when the data is naturally hierarchical or when memory efficiency is critical for large sparse volumes.

### [Sparse Marching Cubes](3825-sparse-marching-cubes.md)
Generates a triangle mesh from a sparse set of 3D points with associated scalar values (e.g., Signed Distance Fields), avoiding the memory overhead of dense 3D grids. It differs from standard Marching Cubes by dynamically gathering corner values for active voxels via lookup, rather than iterating over a fixed dense volume. Choose this when the input is a point cloud or SDF where only a small fraction of the bounding box is occupied.

### [Hierarchical Marching Cubes Enumeration](5223-hierarchical-marching-cubes-enumeration.md)
A recursive octree traversal algorithm that extracts isosurface meshes by enumerating voxel faces, edges, and vertices, delegating the actual geometry generation to a standard Marching Cubes routine at leaf nodes. It differs from [Marching Cubes with Octree](1259-marching-cubes-with-octree.md) and [Sparse Marching Cubes](3825-sparse-marching-cubes.md) by explicitly separating the traversal logic into specialized functions for faces, edges, and vertices to ensure coherent mesh generation across hierarchical boundaries. Choose this when strict manifold continuity across varying octree depths is required without post-processing.

### [2D Stitching](10249-2d-stitching.md)
An index-based tessellation scheme that fills topological gaps between dual grids of different Adaptive Mesh Refinement (AMR) levels to generate crack-free isosurfaces. It merges coarse and fine grid levels seamlessly by generating "stitch cells" (triangles and deformed quadrilaterals) without subdividing existing boundary elements. Unlike constrained Delaunay triangulation, which requires subdividing quadrilateral boundary faces into triangles (causing cracks in 3D Marching Cubes extraction), this method preserves the quadrilateral topology of the grid boundaries. Choose this when working with structured AMR grids where crack-free connectivity between levels is mandatory.

## Background
The need for adaptive grid traversal arises from the limitations of uniform grid isosurface extraction, which suffers from high memory consumption and redundant computation in sparse or multi-scale datasets. Early work on octree-based rendering and extraction laid the groundwork for hierarchical traversal [S1]. The specific enumeration techniques for crack-free hierarchical extraction and the 2D stitching logic for AMR grids were developed to address topological inconsistencies (cracks) that occur when standard algorithms operate on non-uniform grids [S1]. These methods are widely used in scientific visualization, medical imaging, and computational fluid dynamics where data resolution varies significantly across the domain.

## Sources
[S1] CGAL Documentation, "Isosurfacing," https://doc.cgal.org/latest/Isosurfacing/index.html, 2023-10-15
