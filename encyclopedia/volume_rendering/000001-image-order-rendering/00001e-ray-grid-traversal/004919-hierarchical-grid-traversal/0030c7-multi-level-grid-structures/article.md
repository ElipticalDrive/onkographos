# Multi-level Grid Structures

Also called: Hierarchical Grids, Multi-level Grids, Hierarchy of Uniform Grids

## Overview
Multi-level Grid Structures are a family of hierarchical acceleration techniques for ray tracing and volume rendering that organize spatial data into a hierarchy of uniform grids (often arranged as mipmaps or nested blocks) to efficiently skip empty regions and adapt resolution to local density . Unlike flat uniform grids which suffer from excessive memory usage or traversal overhead in sparse scenes, or octrees which may break SIMD coherence, this family uses fixed-depth or adaptive uniform subdivisions to balance memory locality and traversal speed [S1, unofficial]. It sets itself apart from sibling methods like [KD-Tree Partitioned Volume Rendering](1966-kd-tree-partitioned-volume-rendering.md) by maintaining the regular memory access patterns of grid structures while providing hierarchical empty-space skipping.

## Shared principles
All methods in this family share the following mechanism:
1.  **Hierarchical Uniform Subdivision:** The spatial domain is divided into a coarse grid (Level 0). Occupied cells in the coarse grid point to finer sub-grids (Level 1, Level 2, etc.). Empty cells point to null or are skipped entirely.
2.  **Ray-Grid Intersection:** Rays are tested against the Axis-Aligned Bounding Boxes (AABBs) of the grid cells at each level. If a ray does not intersect a coarse cell, the entire sub-grid is skipped.
3.  **Traversal Continuity:** Traversal typically reuses decision variables (DDA steps) between levels to maintain coherence, or computes new intervals based on sorted intersection events .
4.  **Memory Layout:** Data is often stored in flat arrays or linked lists of cells, with occupancy flags or pointers indicating the presence of child grids [S1, unofficial].

## Subtopics

### [Multi-Level Array Grid](3329-multi-level-array-grid.md)
Uses a fixed maximum depth of 2 and a specific heuristic for grid dimensions to balance memory usage and traversal speed.

### [Multi-Grid Ray Traversal](4142-multi-grid-ray-traversal.md)
Explicitly manages multiple grid levels and their intersection events, supporting adaptive step sizing based on cone angles.

### [Two-Level Grids for Ray Tracing on GPUs](4245-two-level-grids-for-ray-tracing-on-gpus.md)
A fixed-depth hierarchical structure optimized for GPU memory constraints, using a single global sort to construct leaf-level references.

### [Macrocell Grid Traversal](5207-macrocell-grid-traversal.md)
Uses a fixed, uniform downsampling factor ($M \times M \times M$) to skip empty regions in bulk, optimized for coherent ray traversal in animated scenes.

### [Multi-resolution grids](12782-multi-resolution-grids.md)
A recursive top-down build approach with hash-table indexed internal nodes, allowing variable resolution across the spatial domain.

### Hierarchical Grid (hgridf) Renderer
#### definition
- [S1, unofficial] A renderer for hierarchical grids (hgridf) that combines primitive generation and rendering in a single pass, updating only 'dirty' cells to support interactive editing.
#### step
- [S1, unofficial] Traverse the top-level grid and visit each grid-cell unless it is empty.
- [S1, unofficial] Check if the current grid-cell is 'dirty' (voxels have changed since the last visit).
- [S1, unofficial] If dirty, use the associated point or polygon engine to regenerate the local isosurface representation for that cell.
- [S1, unofficial] Mark the cell as 'clean' after regeneration.
#### parameter
- [S1, unofficial] The hierarchical grid uses a top-level N×N×N grid and sub-grids of size M×M×M, where M=16.

## Background
The concept of hierarchical grids for volume rendering and ray tracing originated in the early 1990s to address the memory and performance limitations of uniform grids for large datasets. The "hgridf" renderer, described in *Hardware Accelerated Point Rendering of Isosurfaces* (2003), popularized the use of a top-level N×N×N grid with M×M×M sub-grids (typically M=16) to save space and provide constant-time access compared to octrees [S1, unofficial]. The term "hgrid" refers to this specific two-level structure where empty top-level cells are null, and occupied cells contain sub-grids [S1, unofficial]. Later works extended this to multi-level mipmap-like structures for ray marching  and GPU-optimized two-level grids for triangle meshes [S1, unofficial].

## Sources
[S1] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
