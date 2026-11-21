# Surface extraction
Also called: Meshing, Polygonization

## Overview
Surface extraction generates an explicit polygonal mesh from volumetric data, solving the problem of visualizing internal structures or boundaries as geometric primitives rather than rasterized images. This family is distinguished from direct volume rendering methods by producing a discrete geometric representation (mesh) suitable for standard rasterization pipelines, ray tracing, or physical simulation, whereas volume rendering computes pixel colors directly from scalar fields [1-volume-rendering.md].

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid (Cartesian, octree, or tetrahedral) where each cell contains scalar values or occupancy states at its vertices or centers. The core mechanism involves traversing the grid cells and evaluating the data to determine which parts of the domain constitute the surface.

1.  **Domain Traversal**: The algorithm iterates over the volume data structure. For regular grids, this often involves checking each cell or voxel.
2.  **Surface Determination**: A criterion is applied to identify surface locations. In isosurface methods, this involves comparing scalar values against a threshold (isovalue) to detect sign changes or equality [30-isosurface-extraction.md]. In blocky meshing, this involves checking occupancy states or material IDs to identify exposed boundaries [31-blocky-meshing.md].
3.  **Primitive Generation**: Geometric primitives (vertices, edges, faces) are generated based on the determined surface locations.
4.  **Mesh Coherence**: Adjacent primitives are merged or connected to form a coherent mesh, often requiring vertex merging to avoid duplicates and ensure watertightness [30-isosurface-extraction.md].

## Subtopics

### [Isosurface extraction](30-isosurface-extraction.md)
Generates an explicit polygonal mesh representing the surface where a scalar field equals a specific threshold. It interpolates between grid points to create smooth, curved surfaces. Choose this when smooth geometry and topological accuracy are required.

### [Blocky meshing](31-blocky-meshing.md)
Converts discrete volumetric data into polygonal meshes by treating each voxel as a primitive unit, preserving the discrete, axis-aligned nature of the input data. Choose this when performance is critical, the data is inherently discrete (e.g., voxel art, sparse grids), or exact alignment with the grid is desired.

## Background
Surface extraction originated from the need to visualize medical imaging data (CT, MRI) and scientific simulations in a format compatible with standard graphics hardware. The Marching Cubes algorithm, introduced by Lorensen and Cline in 1987, is the foundational method for isosurface extraction on cubic grids [G]. Blocky meshing techniques have evolved alongside voxel-based rendering and sparse volume representations, often leveraging GPU-driven classification and face culling optimizations to handle large datasets [G].

## Sources
[S1] Lorensen, W. E., & Cline, H. E. (1987). Marching cubes: A high resolution 3D surface construction algorithm. SIGGRAPH '87.
