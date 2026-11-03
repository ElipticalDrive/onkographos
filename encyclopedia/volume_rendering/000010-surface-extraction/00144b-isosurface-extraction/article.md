# Isosurface extraction
## Overview
Isosurface extraction generates an explicit polygonal mesh representing the surface where a scalar field equals a specific threshold (isovalue) from volumetric data. It solves the problem of visualizing internal structures or boundaries within volumetric data by approximating the implicit surface $f(x,y,z) = I$. What sets this family apart from direct volume rendering is that it produces an explicit geometric representation (mesh) rather than a rasterized image. Unlike [Blocky meshing](31-blocky-meshing.md), which produces voxel-aligned stair-step surfaces, isosurface methods typically interpolate between grid points to create smooth, curved surfaces [G].

## Shared principles
All isosurface extraction algorithms operate on a discretized domain, typically a regular grid (Cartesian, octree, or tetrahedral) where each cell contains scalar values at its vertices. The core mechanism involves traversing the grid cells and evaluating the scalar field at cell vertices or edges. For cubic grids, vertex values are compared against the isovalue to determine a topological configuration (e.g., a 256-case index). For tetrahedral or hybrid grids, the method identifies edges where the scalar field changes sign. Intersection points are computed using inverse linear interpolation for the parameter and linear interpolation for 3D coordinates: $t = (I - f_a) / (f_b - f_a)$ and $P = V_a + t(V_b - V_a)$. The intersection points are then connected to form primitives (triangles, quads, or polygons). Adjacent cell primitives are merged into a coherent mesh, often requiring vertex merging to avoid duplicates.

## Subtopics

### [Marching Cubes](774-marching-cubes.md)
Generates a triangular mesh from a 3D scalar field by processing each voxel independently. It determines the topology of the isosurface intersection with cube boundaries based on vertex values relative to an isolevel. Uses a fixed lookup table to resolve intersections, making it straightforward but complicated to implement due to fifteen distinct special cases. Choose it for general-purpose cubic grid extraction.

### [Dual Contouring](778-dual-contouring.md)
Places a single vertex inside each grid cell intersected by the surface, optimizing its position by minimizing a Quadratic Error Function (QEF) derived from Hermite data. Unlike Marching Cubes, which places vertices on cell edges, it preserves sharp features and produces manifold, watertight meshes. Choose it when feature preservation and watertightness are critical.

### [Advancing Front Framework for Implicit Surfaces](819-advancing-front-framework-for-implicit-surfaces.md)
Begins with a seed point and grows a triangulation across an implicit surface by advancing the front boundary. Choose it for incremental surface generation or when starting from a known point on the surface.

### [MLS Projection](823-mls-projection.md)
Projects a query point onto a surface defined by input points and normals by fitting a local quadratic function. Choose it for refining vertex positions or projecting points onto implicit surfaces.

### [Isosurface Extraction API](836-isosurface-extraction-api.md)
Provides a standardized interface for invoking various extraction algorithms. Choose it to abstract algorithm selection and manage multiple extraction methods uniformly.

### [VESTA](900-vesta.md)
Extracts isosurfaces from 3D voxel data by constructing a wire frame of N-cycles (closed loops of voxel faces) enclosing active voxels, which are then triangulated. Choose it for volume-enclosing surfaces from voxel data without topological ambiguities.

### [Surface Nets](921-surface-nets.md)
Constructs a mesh by placing a single vertex per active voxel at the weighted average of grid points on intersected edges. Unlike Marching Cubes, it generates exactly one vertex per voxel, resulting in a sparser mesh. Choose it for performance-critical applications where exact vertex placement is less important than speed and sparsity.

### [Extended CLCIR PLIC Reconstruction](954-extended-clcir-plic-reconstruction.md)
Reconstructs a Piecewise Linear Interface Concept (PLIC) plane in Volume of Fluid (VOF) simulations on arbitrary polyhedral grids. Choose it for fluid dynamics simulations requiring accurate interface reconstruction.

### [MISE (Meshing Isosurfaces by Extracting)](977-mise-meshing-isosurfaces-by-extracting.md)
Generates a 3D mesh from a trained DVR model by querying an implicit occupancy field using adaptive resolution. Choose it for neural implicit surface reconstruction.

### [Marching Cubes Lookup Tables](1106-marching-cubes-lookup-tables.md)
Precomputed data structures mapping voxel corner signs to intersected edges and triangle indices. Solves topology determination without runtime branching. Choose it for CPU/GPU implementations prioritizing simplicity and topological consistency over geometric optimality.

### [Convex Contouring with Decision Trees](1201-convex-contouring-with-decision-trees.md)
Produces topologically consistent surfaces with a local convexity property, where the polygonal contour inside each cell encloses a convex negative space. Choose it when topological robustness and convexity guarantees are required.

### [Primal Grid Marching Cubes](1211-primal-grid-marching-cubes.md)
GPU-accelerated technique processing a 3D scalar volume grid using a compute shader. Adapts classic Marching Cubes for parallel execution with fixed-size buffers and atomic counters. Choose it for real-time or high-throughput implicit function to mesh conversion on GPUs.

### [Standard Interval Tree for Isosurface Extraction](1230-standard-interval-tree-for-isosurface-extraction.md)
Data structure to obtain active cells for a particular isosurface with optimal query complexity $O(\log(n) + k)$. Choose it for accelerating cell extraction in large volumes.

### [Octree-based Isosurfacing](1247-octree-based-isosurfacing.md)
Uses an octree as a spatial discretization structure instead of a Cartesian grid. Allows for adaptive resolution but may cause cracks if not adapted to the surface. Choose it for multi-resolution data where adaptive sampling is beneficial.

### [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md)
Approximates the isosurface position between two sample points by iteratively halving the interval based on scalar field values. Choose it for high-precision interpolation in sparse or expensive-to-evaluate fields.

### [High-Dimensional Triangulation Tables](1336-high-dimensional-triangulation-tables.md)
Generates triangulation tables for isosurface extraction in n-dimensional hypercubes. Choose it for extracting surfaces from higher-dimensional data or hypercubic grids.

### [Marching Cubes Smoothing](1594-marching-cubes-smoothing.md)
Smooths a 2D or 3D binary embedding array by assigning negative/positive values to preserve fine details and thin structures. Choose it for post-processing binary embeddings to retain thin features.

### [Subgrid Marching Tetrahedra](1737-subgrid-marching-tetrahedra.md)
Extracts intersection-free isosurfaces using intersections with a tetrahedral grid's edges, guaranteeing freedom from self-intersections. Choose it when topological correctness and intersection-free output are mandatory.

### [Minimal Area Triangulation for Polygons](859-minimal-area-triangulation-for-polygons.md)
Triangulates non-convex or complex polygons resulting from extraction into triangles, minimizing total area to reduce visual artifacts. Choose it for post-processing extracted polygons to improve mesh quality.

### Symmetry Mapping
Maps one of the 256 possible cube configurations to one of 23 canonical base cases using precomputed rotation and base case mappings to simplify triangle generation logic. Choose it to reduce lookup table size and simplify implementation.

### Point Cloud Extraction
Converts isosurfaces to point clouds by evaluating chunks of voxels and extracting vertices at the center of intersected edges, without generating face data. Choose it for point-based rendering or when surface reconstruction is deferred to the GPU.

### Patch Boundary Detection
Detects patch boundaries as groups of rings of linear contours on cell faces for automatic construction of look-up tables. Choose it for automated table generation in custom contouring methods.

### Canonical Enumeration
Sequentially visits all cells in the volume to identify active cells containing the isosurface by checking if the isovalue is contained in the half-open interval $[v_{min}, v_{max})$. Requires only storage for the original volume data and yields linear time complexity $O(n)$. Choose it for simple, memory-efficient extraction when acceleration structures are unnecessary.

## Background
Not given in the sources.

## Sources
[S1] samsfisher/3d-model-xray, https://github.com/samsfisher/3d-model-xray, fetched 2026-10-27 (unofficial)
[S2] https://github.com/swiftcoder/isosurface, https://github.com/swiftcoder/isosurface, fetched 2026-10-26 (unofficial)
[S3] Convex contouring of volumetric data, https://doi.org/10.1007/s00371-003-0216-0, fetched 2026-10-28
[S4] Optimal hybrid memory constrained isosurface extraction, http://nbn-resolving.de/urn:nbn:de:bsz:352-130091, fetched 2026-10-30
