# Mesh Rasterization and Triangle Intersection

Also called: Mesh Voxelization, Triangle Voxelization

## Overview
This family converts continuous triangle meshes into discrete 3D voxel grids, solving the problem of discretizing surface geometry for volume rendering, simulation, or 3D printing [S2, S4, unofficial]. It differs from [Implicit Surface Voxelization](2379-implicit-surface-voxelization.md) by operating on explicit polygonal boundaries rather than analytic functions, and from [Parametric Surface Voxelization](6032-parametric-surface-voxelization.md) by handling arbitrary polygonal topology rather than smooth parametric patches.

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Mesh to Voxel Generation via AABB-Triangle Intersection](3775-mesh-to-voxel-generation-via-aabb-triangle-intersection.md)
Converts mesh to solid voxel grid using AABB-Triangle SAT for shell detection and scanline fill for interior. Choose for deep physical simulations requiring solid occupancy.

### [Mesh Voxelization](6514-mesh-voxelization.md)
Uses ray casting (Möller–Trumbore) to determine inside/outside status, supporting SDF and multi-directional sampling for open meshes. Choose for robust topological handling of complex/concave meshes.

### [Template-based Voxelization](7964-template-based-voxelization.md)
Determines voxel membership via geometric template intersection (6- or 26-connectivity). Choose when strict topological separation properties are required.

## Background
Originating from computer graphics and geometric modeling, these methods bridge the gap between boundary representation (B-Rep) and volumetric data. Early implementations focused on simple ray casting for rendering [S2, S4, unofficial], while later developments emphasized topological correctness for simulation and 3D printing [S1, S3]. The distinction between "shell" (surface) and "solid" (interior) voxelization is critical for applications like heat diffusion or fracture simulation [S1, S3, unofficial].

## Sources
[S1] Mesh Voxelization via Template Intersection, Unofficial Notes, 2023
[S2] Mesh Voxelization via Ray Casting, Unofficial Notes, 2023
[S3] AABB-Triangle Intersection for Voxelization, Unofficial Notes, 2023
[S4] General Mesh Voxelization Practices, Unofficial Notes, 2023
