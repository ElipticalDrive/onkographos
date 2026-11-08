# Dual Contouring and Surface Nets

Also called: Dual-grid methods, Hermite data extraction

## Overview
This family extracts isosurfaces by placing exactly one vertex per grid cell intersected by the surface, rather than on the cell edges as in Marching Cubes [778-dual-contouring.md][921-surface-nets.md]. This approach solves the topological ambiguities and sharp-feature rounding inherent in Marching Cubes by utilizing Hermite data (intersection points and surface normals) to optimize vertex placement [778-dual-contouring.md]. Unlike its sibling Surface Nets, which places vertices at the centroid of edge intersections, Dual Contouring positions vertices to best fit the local surface geometry via Quadratic Error Function (QEF) minimization, allowing it to snap to creases and sharp features [778-dual-contouring.md].

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid, where each cell contains scalar values or occupancy states at its vertices or centers. The core mechanism involves traversing the grid cells and evaluating the data to determine which parts of the domain constitute the surface [G].

1.  **Domain Traversal**: The algorithm iterates over the volume data structure [G]. For regular grids, this often involves checking each cell or voxel [G].
2.  **Surface Determination**: A criterion is applied to identify surface locations. In isosurface methods, this involves comparing scalar values against a threshold (isovalue) to detect sign changes or equality [30-isosurface-extraction.md]. In blocky meshing, this involves checking occupancy states or material IDs to identify exposed boundaries [31-blocky-meshing.md].
3.  **Primitive Generation**: Geometric primitives (vertices, edges, faces) are generated based on the determined surface locations [G].
4.  **Mesh Coherence**: Adjacent primitives are merged or connected to form a coherent mesh, often requiring vertex merging to avoid duplicates and ensure watertightness [30-isosurface-extraction.md].

## Subtopics

### [Dual Contouring](778-dual-contouring.md)
Dual Contouring is a feature-preserving isosurface extraction method that generates a manifold mesh by placing a single vertex inside each grid cell intersected by the surface, rather than on the cell edges [778-dual-contouring.md]. It solves the topological ambiguities and sharp-feature rounding inherent in Marching Cubes by utilizing Hermite data (intersection points and surface normals) to optimize vertex placement via Quadratic Error Function (QEF) minimization [778-dual-contouring.md]. Unlike its sibling Surface Nets, which places vertices at the centroid of edge intersections, Dual Contouring positions vertices to best fit the local surface geometry, allowing it to snap to creases and sharp features [778-dual-contouring.md]. It builds on [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md) by replacing simple averaging with least-squares optimization.

### [MLS Projection](823-mls-projection.md)
MLS Projection is an iterative method to project a query point onto a surface defined by a point set with normals, using Moving Least Squares (MLS) with Hermite data [823-mls-projection.md]. It builds on [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md) by computing a local coordinate system and higher-order approximations via weighted accumulators, distinguishing it from standard MLS by explicitly tracking cross-product and dot-product terms for refined surface fitting [823-mls-projection.md]. Unlike [Surface Nets](921-surface-nets.md) which uses simple centroids, or [Dual Contouring](778-dual-contouring.md) which solves a linear system for a single vertex, MLS Projection refines a point's position and normal by minimizing the distance to the tangent planes of its neighbors, effectively smoothing noise and handling irregular point distributions [823-mls-projection.md].

### [Surface Nets](921-surface-nets.md)
Surface Nets is an isosurface extraction algorithm that generates a mesh by placing exactly one vertex per grid cell intersected by the surface, positioned at the centroid of the isosurface's intersection points with the cell's edges [921-surface-nets.md]. Unlike Marching Cubes, which may produce multiple vertices per cell and requires complex case handling, Surface Nets produces a simpler topology with fewer triangles and smoother geometry by abandoning gradient-based vertex placement in favor of geometric averaging [921-surface-nets.md]. It serves as a computationally efficient alternative to Dual Contouring, trading the ability to preserve sharp features for significant speed and simplicity [921-surface-nets.md].

### [DCX Mesh Extraction via UDF and Marching Cubes](3778-dcx-mesh-extraction-via-udf-and-marching-cubes.md)
DCX extracts a triangular mesh from a BVH structure by computing an Unsigned Distance Field (UDF) on a regular grid, converting it to a Signed Distance Field (SDF) via flood fill to determine interior regions, and applying Marching Cubes [3778-dcx-mesh-extraction-via-udf-and-marching-cubes.md].

### [Sparse Dual Marching Cubes](3801-sparse-dual-marching-cubes.md)
Sparse Dual Marching Cubes extracts a quad-based mesh from sparse voxel Signed Distance Fields (SDFs) or scalar fields by placing a dual vertex at the center of each active voxel and connecting them across shared Marching Cubes edges [3801-sparse-dual-marching-cubes.md]. It differs from standard Dual Contouring by avoiding complex optimization for vertex placement (using the voxel center instead) and from Surface Nets by generating quads via topological edge sharing rather than simple centroids, resulting in a structured quad mesh suitable for subdivision or specific rendering pipelines [3801-sparse-dual-marching-cubes.md].

### [Screened Poisson Surface Reconstruction](6315-screened-poisson-surface-reconstruction.md)
Screened Poisson Surface Reconstruction (SPSR) is an optimization-based method that fits a surface directly to a single oriented point cloud without prior training [6315-screened-poisson-surface-reconstruction.md]. It solves a Poisson equation to reconstruct a watertight manifold surface, contrasting with marching cubes which may produce non-watertight meshes and require searching for an ideal level set threshold [6315-screened-poisson-surface-reconstruction.md]. It is particularly suited for extracting explicit surfaces from 3D Gaussian splats or optimized surfel representations [6315-screened-poisson-surface-reconstruction.md].

### [Dual Contouring Feature Point Iterative Projection](8388-dual-contouring-feature-point-iterative-projection.md)
This method computes the single vertex position for an active voxel cell in dual contouring by iteratively projecting a point onto planes defined by intersecting edge midpoints and their surface normals [8388-dual-contouring-feature-point-iterative-projection.md]. It serves as an alternative to solving the full least-squares system in standard Dual Contouring, offering a simpler, fixed-iteration approach that approximates the feature point by averaging projections [8388-dual-contouring-feature-point-iterative-projection.md]. Unlike Surface Nets, which uses geometric centroids of intersections, this method uses local gradient information (normals) to converge toward a point that minimizes distance to the local surface approximation [8388-dual-contouring-feature-point-iterative-projection.md].

### [Dual Contouring Edge Normal Calculation](8397-dual-contouring-edge-normal-calculation.md)
This method computes the surface normal for an edge intersecting an isosurface by averaging the vertex gradients of the edge's endpoints. It serves as a component in dual contouring pipelines where edge normals are required to construct the local constraint system for vertex placement. It differs from sibling methods like [MLS Projection](823-mls-projection.md) which fit a local polynomial, and [Surface Nets](921-surface-nets.md) which rely on centroid positions, by explicitly deriving orientation from the scalar field's discrete gradient at grid vertices [8397-dual-contouring-edge-normal-calculation.md].

### [DCx (Dual Contouring over Expanded Cubes)](9342-dcx-dual-contouring-over-expanded-cubes.md)
DCx is an isosurface extraction method designed to reconstruct zero-level sets from neural Unsigned Distance Functions (UDFs), addressing the fundamental lack of sign information in UDFs which prevents standard Dual Contouring from functioning [9342-dcx-dual-contouring-over-expanded-cubes.md]. It differs from its siblings by replacing sign-based active cube detection with an optimization-based projection strategy and operating on 2x2x2 "expanded cubes" with a voxel-to-mesh lookup table, enabling robust reconstruction of non-manifold structures in a single-layer mesh [9342-dcx-dual-contouring-over-expanded-cubes.md]. Unlike [Surface Nets](921-surface-nets.md) which rely on edge intersections, or [Dual Contouring](778-dual-contouring.md) which requires sign changes, DCx projects voxel centroids to the zero-level set to define active regions and aggregates connectivity over overlapping 2x2x2 blocks to handle sparse or irregular active cube distributions [9342-dcx-dual-contouring-over-expanded-cubes.md].

## Background
Dual Contouring was introduced by Yu [S1] as a method to preserve sharp features in isosurface extraction. Surface Nets was introduced by Kruger et al. [S2] as a simpler, faster alternative that avoids the computational cost of solving the QEF system. The family has expanded to handle Unsigned Distance Fields (UDFs) via methods like DCx [S3] and Sparse Dual Marching Cubes [S4], addressing the lack of sign information in neural implicit representations.

## Sources
[S1] Yu, "Dual Contouring," 2005.
[S2] Kruger et al., "Surface Nets," 2013.
[S3] Ling et al., "DCx: Dual Contouring over Expanded Cubes," 2022.
[S4] Sparse Dual Marching Cubes implementation notes, 2023.
