# Neural and Learning-based Extraction

Also called: Neural Isosurfacing, Differentiable Meshing

## Overview
This family extracts triangle meshes from neural implicit representations (occupancy fields, SDFs, or NeRF densities) by combining neural query capabilities with geometric extraction algorithms. It solves the problem of converting continuous, learned volumetric functions into explicit polygonal meshes suitable for rendering or simulation. Unlike static methods like [Marching Cubes](30-isosurface-extraction.md) on fixed grids, this family distinguishes itself by either adaptively querying the neural network to save computation (e.g., MISE) or by making the extraction process fully differentiable to allow gradient-based optimization of the mesh geometry (e.g., FlexiCubes).

## Shared principles
All methods in this family operate on a discretized domain, typically a regular grid (Cartesian, octree, or tetrahedral) where each cell contains scalar values or occupancy states at its vertices or centers. The core mechanism involves traversing the grid cells and evaluating the data to determine which parts of the domain constitute the surface.

1.  **Domain Traversal**: The algorithm iterates over the volume data structure. For regular grids, this often involves checking each cell or voxel.
2.  **Surface Determination**: A criterion is applied to identify surface locations. In isosurface methods, this involves comparing scalar values against a threshold (isovalue) to detect sign changes or equality [30-isosurface-extraction.md]. In blocky meshing, this involves checking occupancy states or material IDs to identify exposed boundaries [31-blocky-meshing.md].
3.  **Primitive Generation**: Geometric primitives (vertices, edges, faces) are generated based on the determined surface locations.
4.  **Mesh Coherence**: Adjacent primitives are merged or connected to form a coherent mesh, often requiring vertex merging to avoid duplicates and ensure watertightness [30-isosurface-extraction.md].

## Subtopics

### [MISE (Meshing Isosurfaces by Extracting)](977-mise-meshing-isosurfaces-by-extracting.md)
MISE is an adaptive meshing algorithm that generates a 3D triangle mesh from a trained differentiable volumetric rendering (DVR) model by querying an implicit occupancy field. It solves the problem of extracting high-fidelity surfaces from neural implicit representations without requiring a uniformly dense voxel grid, which is computationally prohibitive for high-resolution outputs. Unlike static marching cubes on a fixed grid, MISE adaptively refines the query points around the isosurface, concentrating computational effort where the occupancy function changes most rapidly.

### [FlexiCubes](3669-flexicubes.md)
FlexiCubes is a differentiable isosurface extraction method that converts a Signed Distance Field (SDF) defined on a voxel grid into a triangle mesh, specifically designed for gradient-based optimization of geometric, visual, or physical objectives. Unlike static extraction methods, it introduces learnable per-cube weights (beta, alpha, gamma) and vertex deformations to resolve topological ambiguities and improve mesh quality during training. It builds on [Neural and Learning-based Extraction](6371-neural-and-learning-based-extraction.md) by providing a fully differentiable pipeline where the mesh geometry itself becomes an optimization variable.

### [NeRF Density Field Mesh Extraction](4104-nerf-density-field-mesh-extraction.md)
Extracts a triangle mesh from a trained Neural Radiance Field (NeRF) by evaluating the learned volume density $\sigma$ on a 3D grid and applying the marching cubes algorithm at a specific threshold. It differs from adaptive methods like MISE [MISE (Meshing Isosurfaces by Extracting)](977-mise-meshing-isosurfaces-by-extracting.md) by using a fixed uniform grid, which is simpler to implement but less efficient for complex geometries, and from differentiable methods like FlexiCubes [FlexiCubes](3669-flexicubes.md) by producing a static mesh without gradient-based optimization of the surface vertices [Parent](6371-neural-and-learning-based-extraction.md).

## Background
The extraction of meshes from neural implicit representations emerged with the rise of Neural Radiance Fields (NeRF) [S1, unofficial]. Early approaches relied on standard marching cubes on fixed grids, which were computationally expensive for high resolutions. MISE [MISE (Meshing Isosurfaces by Extracting)](977-mise-meshing-isosurfaces-by-extracting.md) introduced adaptive querying to address this efficiency bottleneck. Subsequently, methods like FlexiCubes [FlexiCubes](3669-flexicubes.md) extended the field by making the extraction process differentiable, enabling end-to-end optimization of 3D shapes from images or other signals [S2, unofficial].

## Sources
[S1] MISE and NeRF Extraction Notes, URL, date fetched
[S2] FlexiCubes Notes, URL, date fetched
