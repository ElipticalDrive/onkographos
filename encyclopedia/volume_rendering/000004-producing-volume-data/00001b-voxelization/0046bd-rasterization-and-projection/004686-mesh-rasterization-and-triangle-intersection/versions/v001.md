# Mesh Rasterization and Triangle Intersection

Also called: Geometric Voxelization, Triangle-Based Voxelization

## Overview
This family converts continuous triangular meshes into discrete 3D voxel grids by evaluating the geometric relationship between the mesh primitives and the voxel domain [S1, S2, S3, S4]. It solves the problem of digitizing polygonal models for volume rendering or simulation by determining voxel occupancy through explicit intersection tests rather than projection [S1, S2, S3, S4]. It differs from [Rasterization-based Voxelization](11755-rasterization-and-gpu-shaders.md) by operating in world space using geometric intersection tests rather than projecting geometry onto screen-space tiles [S1, S2, S3, unofficial], and from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) which operates on discrete samples rather than continuous geometry [S3, unofficial].

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Mesh to Voxel Generation via AABB-Triangle Intersection](3775-mesh-to-voxel-generation-via-aabb-triangle-intersection.md)
This method converts a triangle mesh into a solid 3D voxel grid by first identifying surface voxels through Axis-Aligned Bounding Box (AABB) vs. Triangle intersection tests, and then filling the interior volume using a scanline fill algorithm. It differs from [Fake Voxelization via Triangle-Cube Intersection](6951-fake-voxelization-via-triangle-cube-intersection.md) by producing a solid, watertight volume rather than a hollow surface shell, and from [Rasterization-based Voxelization](908-rasterization-based-voxelization.md) by operating in world space using geometric intersection tests rather than projecting geometry onto screen-space tiles.

### [Mesh Voxelization](6514-mesh-voxelization.md)
Mesh Voxelization converts a continuous triangular surface mesh (e.g., STL, OBJ) into a discrete 3D voxel grid by determining the occupancy of each grid cell via ray-mesh intersection tests. Unlike [Rasterization-based Voxelization](908-rasterization-based-voxelization.md), which projects geometry onto a 2D plane and extrudes coverage, this method evaluates the topological relationship (inside/outside) of the voxel center relative to the closed surface using the ray casting parity rule. It serves as the foundational technique for generating solid or surface-only voxel representations from polygonal models, distinct from [Point Cloud Voxelization](1000-point-cloud-voxelization.md) which operates on discrete samples rather than continuous geometry.

### [Template-based Voxelization](7964-template-based-voxelization.md)
Template-based Voxelization computes the digitization of a triangle or a mesh by determining if a voxel belongs to the digitization based on whether a template centered at the voxel intersects the triangle. It is a CPU straightforward implementation of reference [79] (or [78] in earlier versions). This method is distinct from rasterization-based approaches by using geometric intersection tests with predefined connectivity templates rather than projecting primitives onto a screen or grid.

## Background
The techniques described herein rely on fundamental computational geometry algorithms for triangle intersection, such as the Separating Axis Theorem (SAT) [S3, unofficial] and the Möller–Trumbore algorithm [S2, unofficial]. The ray-casting parity rule for determining inside/outside status is a standard topological test in computer graphics [S4, unofficial]. Template-based voxelization is based on digital geometry theory, specifically the work on separating digitization [S1, S2]. Specific implementations and optimizations for mesh voxelization are documented in various unofficial sources and technical notes [S1, S2, S3, S4, unofficial].

## Sources
[S1] Mesh Voxelization Implementation Notes, URL, 2023
[S2] Ray-Triangle Intersection and Voxelization Techniques, URL, 2023
[S3] AABB-Triangle Intersection for Voxelization, URL, 2023
[S4] Mesh Voxelization Algorithm Details, URL, 2023
