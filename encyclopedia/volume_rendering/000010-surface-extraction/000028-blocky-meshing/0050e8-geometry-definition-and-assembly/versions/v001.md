# Geometry Definition and Assembly

Also called: Voxel Mesh Construction, Blocky Geometry Pipeline

## Overview
Geometry Definition and Assembly is the family of methods responsible for converting discrete 3D voxel occupancy data into contiguous, render-ready polygonal triangle meshes. It solves the problem of rendering large, blocky volumes efficiently by culling internal faces between adjacent solid voxels, thereby reducing vertex count and draw calls compared to naive voxel rendering [S1, S9, unofficial]. Unlike smooth surface extraction methods like Marching Cubes, this family produces a blocky, axis-aligned aesthetic suitable for voxel-art styles [S1, unofficial][S8]. It differs from smooth extraction by emitting axis-aligned quads or triangles at voxel boundaries rather than interpolating geometry through the volume [S1, unofficial].

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

### [Block face vertex layout](2374-block-face-vertex-layout.md)
Defines the explicit geometric structure of a single unit voxel cube, specifying positions, normals, colors, and texture coordinates for its six faces. It provides the foundational atomic geometry upon which blocky meshing techniques build, supporting per-face attributes like distinct ambient occlusion colors or texture tiling via vertex duplication at corners.

### [Mesh Assembly and Data Structure](6268-mesh-assembly-and-data-structure.md)
Responsible for aggregating raw geometric primitives into contiguous, render-ready buffer structures suitable for GPU submission. It solves the problem of fragmented geometry and material heterogeneity by consolidating vertices, indices, and attributes from disparate voxel blocks into dense memory regions, optimizing bandwidth and reducing draw call overhead.

### [Voxel Face and Geometry Generation](11724-voxel-face-and-geometry-generation.md)
The family of algorithms that convert discrete 3D voxel grids into polygonal triangle meshes by culling internal faces between adjacent solid voxels. It differs from smooth extraction by emitting axis-aligned quads or triangles at voxel boundaries rather than interpolating geometry through the volume.

## Background
The approach originates from the requirements of voxel-based games and simulations where performance on large, dynamic grids is critical. The specific optimization of face culling to reduce geometry count is a standard technique in voxel rendering [S1, S9, unofficial]. The aesthetic choice of axis-aligned primitives over smooth surfaces defines the "blocky" style distinct from isosurface methods [S1, unofficial][S8].

## Sources
[S1] Voxel Face Culling Logic, URL, 2023-10-01
[S2] Voxel Vertex Emission Pattern, URL, 2023-10-01
[S3] Translucent Voxel Face Rules, URL, 2023-10-01
[S5] Voxel Grid Padding Strategy, URL, 2023-10-01
[S7] Triangle Winding Convention, URL, 2023-10-01
[S8] Blocky Aesthetic Definition, URL, 2023-10-01
[S9] Chunk Boundary Handling, URL, 2023-10-01
