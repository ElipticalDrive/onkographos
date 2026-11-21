# Surface extraction
## Overview
[G] Surface extraction generates an explicit polygonal mesh from volumetric data, solving the problem of visualizing internal structures or boundaries as geometric primitives rather than rasterized images. This family produces a discrete geometric representation suitable for standard rasterization pipelines, ray tracing, or physical simulation, contrasting with direct volume rendering which computes pixel colors directly from scalar fields [G].

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid (Cartesian, octree, or tetrahedral) where each cell contains scalar values or occupancy states at its vertices or centers [G]. The core mechanism involves traversing the grid cells and evaluating the data to determine which parts of the domain constitute the surface [G].
1. **Domain Traversal**: The algorithm iterates over the volume data structure [G]. For regular grids, this often involves checking each cell or voxel [G].
2. **Surface Determination**: A criterion is applied to identify surface locations [G]. In [Isosurface extraction](30-isosurface-extraction.md), this involves comparing scalar values against a threshold (isovalue) to detect sign changes or equality. In [Blocky meshing](31-blocky-meshing.md), this involves checking occupancy states or material IDs to identify exposed boundaries.
3. **Primitive Generation**: Geometric primitives (vertices, edges, faces) are generated based on the determined surface locations [G].
4. **Mesh Coherence**: Adjacent primitives are merged or connected to form a coherent mesh, often requiring vertex merging to avoid duplicates and ensure watertightness [G].

## Subtopics

### [Isosurface extraction](30-isosurface-extraction.md)
Generates an explicit polygonal mesh representing the surface where a scalar field equals a specific threshold. It interpolates between grid points to create smooth, curved surfaces. It is chosen when smooth geometry and topological accuracy are required.

### [Blocky meshing](31-blocky-meshing.md)
Converts discrete volumetric data into polygonal meshes by treating each voxel as a primitive unit, preserving the discrete, axis-aligned nature of the input data. It is chosen when performance is critical, the data is inherently discrete (e.g., voxel art, sparse grids), or exact alignment with the grid is desired.

## Background
Surface extraction originated from the need to visualize medical imaging data (CT, MRI) and scientific simulations in a format compatible with standard graphics hardware [G]. The Marching Cubes algorithm, introduced by Lorensen and Cline in 1987, is the foundational method for isosurface extraction on cubic grids [G]. Blocky meshing techniques have evolved alongside voxel-based rendering and sparse volume representations, often leveraging GPU-driven classification and face culling optimizations to handle large datasets [G].

## Sources
Not given in the sources.
