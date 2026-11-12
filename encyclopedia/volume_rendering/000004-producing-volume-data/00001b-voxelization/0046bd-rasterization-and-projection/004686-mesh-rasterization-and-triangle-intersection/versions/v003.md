# Mesh Rasterization and Triangle Intersection

Also called: Triangle Voxelization, Mesh-to-Voxel Conversion

## Overview
This family converts continuous 3D triangle meshes into discrete volumetric representations (voxel grids) by determining the occupancy or geometric relationship of grid cells relative to the mesh surface [S1, S3]. It solves the problem of discretizing surface geometry for applications such as volume rendering, physical simulation, and 3D printing preparation. Unlike sibling methods that may use implicit functions or parametric projections, this family explicitly relies on geometric intersection tests between triangles and voxel primitives (AABBs, templates, or ray-casting volumes) to determine voxel states [S2, S4, unofficial].

## Shared principles
All methods in this family map a conceptual or physical source into a discrete or analytic volumetric representation suitable for rendering [G]. The core mechanism involves:
1. **Domain Definition**: Establishing a spatial domain, typically a 3D grid with resolution $(N_x, N_y, N_z)$ and bounding box, or an infinite analytic space [G].
2. **Value Assignment**: Determining the value (density, distance, color) for each point in the domain based on the source method (geometric intersection, function evaluation, or physical state) [G].
3. **Storage/Access**: Either storing the result in a discrete array (voxelization, simulation) or providing an interface to compute it on-the-fly (procedural) [G].

## Subtopics

### [Mesh to Voxel Generation via AABB-Triangle Intersection](3775-mesh-to-voxel-generation-via-aabb-triangle-intersection.md)
Converts a mesh into a solid, non-hollow voxel grid using AABB-Triangle intersection tests based on the Separating Axis Theorem (SAT) followed by a scanline fill for interior parity. Choose this for deep physical simulations requiring solid occupancy.

### [Mesh Voxelization](6514-mesh-voxelization.md)
Uses ray casting (Möller–Trumbore) to determine inside/outside status, supporting multi-directional sampling for open meshes and optional Signed Distance Field (SDF) computation. Choose this for robust topological handling of complex or non-watertight meshes.

### [Template-based Voxelization](7964-template-based-voxelization.md)
Determines voxel membership by testing if a geometric template (defined by 6- or 26-connectivity separation properties) intersects the triangle. Choose this when strict topological separation (no path crossing the boundary) is required.

### [Triangle Rendering with Thickness](12908-triangle-rendering-with-thickness.md)
Renders a triangle as a volumetric shell by voxelizing points within a specified distance from the triangle's plane or edges. Choose this for thin-shell rendering or anti-aliasing effects.

## Background
The conversion of polygonal meshes to voxel grids is a fundamental operation in computer graphics and computational geometry. Early methods focused on simple rasterization techniques, while modern approaches emphasize topological correctness and robustness against floating-point errors [S2, S4, unofficial]. The AABB-Triangle intersection approach is widely used in game engines for collision detection and voxel-based physics [S3, unofficial]. Ray-casting methods are standard in CAD and 3D printing pipelines for generating watertight volumes [S4, unofficial]. Template-based methods originate from digital topology research, ensuring that digitized surfaces maintain specific connectivity properties [S1, S3].

## Sources
[S1] CGAL Documentation, "Digital Surface Extraction," https://doc.cgal.org/latest/Surface_mesh_simplification/index.html, 2023-10-01
[S2] Voxelizer-rs Documentation, "Mesh Voxelization," https://github.com/robert-ancell/voxelizer-rs, 2023-10-01
[S3] Unofficial Implementation Notes, "AABB Triangle Intersection," 2023-10-01
[S4] Unofficial Implementation Notes, "Ray Casting Voxelization," 2023-10-01
