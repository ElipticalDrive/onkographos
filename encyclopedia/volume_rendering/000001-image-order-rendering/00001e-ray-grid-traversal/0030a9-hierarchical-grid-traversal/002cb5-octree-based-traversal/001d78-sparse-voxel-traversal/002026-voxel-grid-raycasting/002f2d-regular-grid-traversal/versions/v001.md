# Regular Grid Traversal

Also called: 3D-DDA, Voxel Grid Traversal, Sparse Voxel Grid Traversal

## Overview
Regular Grid Traversal accelerates ray-scene intersection by stepping through a uniform 3D grid of cells, skipping empty space without primitive tests [S1, S2, unofficial]. It differs from hierarchical methods (e.g., [Sparse Voxel Traversal](5228-sparse-voxel-tracing.md)) by using a fixed-size array, offering high cache efficiency on GPUs for uniform data but requiring $O(N^3)$ memory [S3, unofficial].

## Shared principles
All children use a 3D Digital Differential Analyzer (3D-DDA) to traverse the grid. The ray origin is mapped to an initial cell $(X, Y, Z)$. Distances to the next grid boundary in each axis are tracked via $tMax$ and $tDelta$. $tDelta$ is constant per axis: $tDeltaX = dx / |dir_x|$ [S1, S2, unofficial]. $tMax$ is updated by adding $tDelta$ when the ray crosses a boundary. The ray advances to the nearest boundary ($min(tMaxX, tMaxY, tMaxZ)$) [S1, S2, unofficial].

## Subtopics

### [Regular Grid](3950-regular-grid.md)
Uses a 1D array of linked lists for primitives. Conservative mapping ensures no primitive is missed. Handles intersections outside the current cell by continuing traversal.

### [Voxel-based Ray Tracing for GI and Specular Reflections](4091-voxel-based-ray-tracing-for-gi-and-specular-reflections.md)
Uses a 128³ occupancy grid for GI/Reflection fallbacks. Integrates with Light Propagation Volumes (LPV) and requires `GL_ARB_shader_image_load_store`.

### [Optimised Voxel Ray Marching](4240-optimised-voxel-ray-marching.md)
Uses bitwise operations for chunk indexing to minimize arithmetic overhead. Optimized for fixed-step traversal within working chunks.

## Background
Regular grids are a foundational spatial acceleration structure in ray tracing [S1, S2, unofficial]. They partition space into disjoint sub-blocks, unlike BVH which partitions primitives [S2, S3, unofficial]. Effective on GPUs due to spatial locality [S3, unofficial].

## Sources
[S1] Voxel-based Ray Tracing for GI and Specular Reflections notes
[S2] Optimised Voxel Ray Marching notes
[S3] Regular Grid notes
