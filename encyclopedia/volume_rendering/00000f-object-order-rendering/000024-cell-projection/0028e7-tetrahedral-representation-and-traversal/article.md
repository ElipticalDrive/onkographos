# Tetrahedral representation and traversal

Also called: Projected Tetrahedra, Tetrahedral Volume Rendering, Marching Tetrahedrons

## Overview
The tetrahedral representation and traversal family decomposes volumetric data into tetrahedral primitives to leverage hardware-accelerated rasterization or efficient geometric traversal, solving the problem of rendering unstructured or irregular grids without the sampling artifacts of grid-based methods [S1][S2, unofficial][S3]. Unlike [Cell projection](26-cell-projection.md) methods that may use cubes or splats, this family specifically utilizes tetrahedra to resolve topological ambiguities in isosurface extraction or to provide exact geometric projection for transparency [S1][S2, unofficial][S3]. It differs from [Projected Tetrahedra](877-projected-tetrahedra.md) in that the broader family includes traversal-based ray casting and isosurface extraction, whereas PT is strictly an object-order rasterization technique.

## Shared principles
The tetrahedral family shares the fundamental mechanism of decomposing volumetric cells into tetrahedra to handle irregular connectivity and topological consistency [S1][S2, unofficial][S3].

1.  **Decomposition:** Volumetric cells (typically cubes or hexahedra) are subdivided into tetrahedra. For cubes, this often involves splitting along a main diagonal into 5 or 6 tetrahedra to ensure consistent connectivity and avoid cracks between adjacent cells [S1][S2, unofficial][S3].
2.  **Geometric Primitive:** The tetrahedron is the primary primitive. It is defined by four vertices $\{V_0, V_1, V_2, V_3\}$ and associated scalar values or attributes [S1][S2, unofficial][S3].
3.  **Topological Consistency:** Using tetrahedra resolves the topological ambiguities inherent in cube-based methods (like Marching Cubes) because the intersection of an isosurface with a tetrahedron is always a simple polygon, eliminating complex case tables [S1].
4.  **Projection/Traversal:** The tetrahedra are either projected onto the 2D image plane for rasterization (object-order) or traversed by rays for ray casting (image-order) [S2, unofficial][S3].

## Subtopics

### [Tetrahedral Volume Rendering](820-tetrahedral-volume-rendering.md)
A GPU-based method using a two-pass pipeline to project tetrahedra and compute intersection data, supporting Basis Tensor volumes for isosurface rendering via density addition.

### [Projected Tetrahedra](877-projected-tetrahedra.md)
An object-order technique that projects tetrahedra into 2D triangles, classifies them into projection classes, and rasterizes them in back-to-front order to handle transparency.

### [Marching Tetrahedrons Algorithm](940-marching-tetrahedrons-algorithm.md)
An isosurface extraction method that decomposes cubes into tetrahedra to guarantee topological consistency, resolving ambiguities present in Marching Cubes.

### [Adaptive Delaunay Update](4923-adaptive-delaunay-update.md)
A strategy for maintaining Delaunay validity during optimization by performing periodic topological flips (Pachner moves) when geometric constraints are violated.

### [Parallel Projected Tetrahedra with KD-Tree Partitioning](8767-parallel-projected-tetrahedra-with-kd-tree-partitioning.md)
A parallel rendering approach that partitions unstructured grids using a kd-tree and uses sort-last compositing to render large datasets.

### [Tetrahedral Mesh Connectivity Encoding](8957-tetrahedral-mesh-connectivity-encoding.md)
A compression scheme using the Basic Cut-Border Machine (CBM) to encode tetrahedral mesh connectivity via a stream of opcodes during region-growing traversal.

### Point Representation via Tetrahedra
*   **Definition:** Representing data points as small tetrahedra to fit them into the triangular mesh framework for rendering [S1].
*   **Parameter:** `size`: Specifies the size of the point tetrahedra relative to the data ranges [S1].

### Grid-based Traversal
*   **Definition:** A method using a hierarchical data structure for rectilinear grids to quickly determine tetrahedra possibly hit by traversed grid cells, enabling interactive rendering of irregular volume data [S2, unofficial].
*   **Step:** Attach all tetrahedra intersecting a given grid cell to that cell. Use minimum and maximum values to cull regions outside of the current iso-value [S2, unofficial].
*   **Number:** Rendered a bioelectric field of over one million tetrahedra at interactive rates (512x512 viewport) with 16 processors or more [S2, unofficial].

### Delaunay Triangulation for Tetrahedral Representation
*   **Step:** Obtain a tetrahedral representation of the Voronoi sites via Delaunay triangulation using gDel3D to accelerate computation on the GPU. Recompute the triangulation from scratch at every iteration [S3].

## Background
The use of tetrahedra in volume rendering and geometric processing originated from the need to handle unstructured grids and resolve topological issues in isosurface extraction. The Marching Tetrahedrons Algorithm (MTA) was developed to address the topological ambiguities of Marching Cubes by ensuring consistent surface connectivity [S1]. Interactive rendering of large unstructured tetrahedral datasets was pioneered by Parker et al., who demonstrated ray tracing of over one million tetrahedra on supercomputers using grid-based traversal [S2, unofficial]. More recently, GPU-accelerated projected tetrahedra methods have enabled real-time rendering of unstructured data by leveraging hardware rasterization [S3].

## Sources
[S1] Computing and Displaying Isosurfaces in R, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S3] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
