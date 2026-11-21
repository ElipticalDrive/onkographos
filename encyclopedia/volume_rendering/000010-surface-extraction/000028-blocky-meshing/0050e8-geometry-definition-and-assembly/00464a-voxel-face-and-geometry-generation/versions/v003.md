# Voxel Face and Geometry Generation

Also called: Voxel Face Meshing, Blocky Surface Extraction

## Overview
Voxel Face and Geometry Generation is a family of surface extraction algorithms that convert a discrete 3D voxel grid (occupancy or density field) into a triangular mesh by emitting geometry only for voxel faces exposed to empty space or chunk boundaries [S1, S3, S5, unofficial]. It solves the problem of rendering large voxel worlds efficiently by culling internal faces between adjacent solid voxels, thereby reducing vertex count and draw calls compared to naive voxel rendering [S1, S9, unofficial]. Unlike smooth surface extraction methods like Marching Cubes, this method produces a blocky, axis-aligned aesthetic suitable for voxel-art styles [S1, unofficial][S8]. It differs from [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by focusing on simple neighbor-based face culling rather than combining multiple voxel sets via boolean operations [S1, unofficial].

## Shared principles
The family operates on a 3D occupancy grid or density field. The core mechanism involves iterating through voxels and determining which faces are "exposed" to empty space or a chunk boundary.

1.  **Neighbor Check:** For each solid voxel, the algorithm checks its 6 immediate neighbors (±X, ±Y, ±Z) [S1, S9, unofficial].
    *   If a neighbor is empty (Air, density below threshold, or `visible=false`), the face is exposed [S1, S5, unofficial].
    *   If a neighbor is out of chunk bounds, it is treated as empty/visible [S9, unofficial].
    *   *Translucency Rule:* For translucent voxels, a face is generated if the neighbor is empty; if the neighbor is opaque, the face is hidden [S3, unofficial].
2.  **Face Culling:** Internal faces between two adjacent solid voxels are discarded. This is the primary optimization that reduces geometry count [S1, S9, unofficial].
3.  **Vertex/Quad Emission:** For each exposed face, a quad (two triangles) is generated. The vertices are calculated by adding the voxel's world position (scaled by `BlockSize`) to base quad vertices defined for the face direction [S2, S7, unofficial].
4.  **Winding Order:** Triangle winding must be consistent (e.g., counter-clockwise) to ensure correct back-face culling [S2, S7, unofficial]. A common pattern is `(0,1,2)` and `(3,2,1)` relative to the current vertex count [S2, S7, unofficial].
5.  **Padding:** To simplify boundary checks, the voxel grid is often conceptually or physically padded with a constant empty boundary on all sides [S1, S3, S5, unofficial].

## Subtopics

### [Boolean Voxel Meshing](971-boolean-voxel-meshing.md)
Boolean Voxel Meshing converts a 3D boolean voxel occupancy grid into a triangular mesh by detecting faces and vertices at voxel boundaries. It solves the problem of generating a watertight, manifold representation of a discrete binary volume, producing a "blocky" or Minecraft-like aesthetic. It differs from [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md) by explicitly computing shared vertex indices across adjacent voxels to merge coplanar faces into a single manifold, rather than emitting independent quads for every visible face. It differs from [VoxelGrid Mesh Generation](2714-voxelgrid-mesh-generation.md) by sharing vertices between adjacent voxels to produce a manifold mesh, whereas the latter treats each voxel as an independent object resulting in a non-manifold mesh with duplicated geometry.

### [Voxel Face Mesh Generation](4212-voxel-face-mesh-generation.md)
Voxel Face Mesh Generation is a surface extraction algorithm that constructs a triangle mesh from a 3D voxel grid by emitting a single unit quad for every face of a solid voxel that is exposed to empty space or a chunk boundary. It solves the problem of rendering large voxel worlds efficiently by culling internal faces between adjacent solid voxels, thereby reducing vertex count and draw calls compared to naive voxel rendering. Unlike smooth surface extraction methods like Marching Cubes, this method produces a blocky, axis-aligned aesthetic suitable for voxel-art styles. It differs from [Boolean Voxel Meshing](971-boolean-voxel-meshing.md) by focusing on simple neighbor-based face culling rather than combining multiple voxel sets via boolean operations.

### Classic Meshing
#### definition
- [S1, unofficial] A mesh generation technique where each voxel face is generated individually, resulting in higher vertex and index counts than greedy meshing.
#### layout
- [S1, unofficial] Functions `addBackFace`, `addFrontFace`, `addLeftFace`, `addRightFace`, `addTopFace`, `addBottomFace` take a single voxel coordinate `x, y, z` and append vertices to `std::vector<PackedVertex>& vertices` and indices to `std::vector<GLuint>& indices`.

### Self-bootstrap Geometry Generation
#### step
- [S2] Generate initial geometry purely from the model itself using 8 coarse tri-vectors.
- [S2] Do not use existing modules from previous work for geometry generation.
#### number
- [S2] Maintains high PSNR even with a much worse early-stopped RGBA reconstruction (Fig. 6).
#### comparison
- [S2] Achieves robust high PSNR across various initial geometry choices, proving quality stems from the Strivec representation rather than the initial geometry.

### Voxel Outline Generation
#### definition
- [S3, unofficial] Generating line outlines for voxel joints using the same 2D color-planes used for face merging.
#### step
- [S3, unofficial] Generate lines around the perimeter and holes of the rectangular faces in the 2D planes.
- [S3, unofficial] When rendering lines, apply a slight depth buffer bias to avoid z-fighting.
#### pitfall
- [S3, unofficial] This process generates duplicate line segments (exactly 2x), which are not filtered out.

## Background
The concept of extracting meshes from voxel grids by culling internal faces is foundational to voxel-based games like Minecraft, where efficiency is critical for rendering large worlds [S1, S9, unofficial]. The specific implementation details, such as padding grids to handle boundaries and using lookup tables for texture atlas UVs, are standard practices in engine development [S2, S7, unofficial]. The "Self-bootstrap" variant originates from the Strivec paper, which uses coarse geometry for pruning rather than final rendering [S2]. The "Voxel Outline" technique is described in Fogleman's voxel rendering blog, focusing on aesthetic line art generation [S3, unofficial].

## Sources
[S1] Justinj68/OM3D_PROJECT, https://github.com/Justinj68/OM3D_PROJECT, fetched 2026-10-27 (unofficial)
[S2] Strivec: Sparse Tri-Vector Radiance Fields, https://doi.org/10.1109/iccv51070.2023.01611, fetched 2026-10-26
[S3] Voxel Rendering Techniques - Medium, https://medium.com/@fogleman/voxel-rendering-techniques-fa8d869457ca, fetched 2026-10-27 (unofficial)
