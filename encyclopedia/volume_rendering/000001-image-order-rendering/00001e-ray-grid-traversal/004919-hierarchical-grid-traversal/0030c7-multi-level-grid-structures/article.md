# Multi-level Grid Structures

Also called: Hierarchical Grids, Multi-level Grids, Hierarchy of Grids

## Overview
Multi-level Grid Structures are a family of hierarchical acceleration techniques for ray tracing and volume rendering that organize spatial data into nested grids of varying resolution [S1, unofficial][S2]. They solve the problem of inefficient traversal in sparse or non-uniform scenes by allowing rays to skip large empty regions at coarse levels before descending into finer grids only where necessary [S1, unofficial][S2]. Unlike flat grids, they adapt resolution based on object density, and unlike pointer-based trees (e.g., [Octree-based traversal](5348-octree-based-traversal.md)), they often use contiguous array-based grids or fixed-depth hierarchies to optimize for memory locality and reduced ray-primitive intersection counts [S1, unofficial][S2].

## Shared principles
The family shares the mechanism of recursive or fixed-depth spatial decomposition using grid structures.

1.  **Hierarchical Decomposition**: The scene or volume is divided into a hierarchy of grids. A coarse "top-level" grid covers the entire bounding box. Occupied cells in the top-level grid point to finer "child" grids [S1, unofficial][S2].
2.  **Ray-Grid Traversal**: Rays traverse the hierarchy starting from the root. If a ray intersects an empty cell in a parent grid, the entire subtree is skipped [S1, unofficial][S2]. If the cell is occupied, the ray descends to the child grid [S1, unofficial][S2].
3.  **Grid Representation**: Child grids are typically uniform grids (e.g., $M \times M \times M$) rather than arbitrary subdivisions [S1, unofficial][S2]. This allows for fast traversal using integer arithmetic and DDA (Digital Differential Analyzer) [S1, unofficial][S2].
4.  **Occupancy Flags**: Cells in coarser grids often store simple occupancy flags (boolean) indicating whether any primitive or voxel exists in the corresponding region of the finer grid [S1, unofficial][S2].

## Subtopics

### [Multi-Level Array Grid](3329-multi-level-array-grid.md)
Uses contiguous array-based grids with variable depth. Choose when memory locality is critical and scene density varies significantly.

### [Multi-Grid Ray Traversal](4142-multi-grid-ray-traversal.md)
Traverses multiple binary occupancy grids in parallel. Choose for dense volumetric data with disjoint regions.

### [Two-Level Grids for Ray Tracing on GPUs](4245-two-level-grids-for-ray-tracing-on-gpus.md)
Fixed-depth two-level hierarchy optimized for GPU. Choose for dynamic scenes with non-uniform primitive distributions.

### [Macrocell Grid Traversal](5207-macrocell-grid-traversal.md)
Two-level hierarchy with fixed macrocell size. Choose for animated scenes with medium-sized grids where build overhead must be low.

### Hierarchical Grid (hgridf) Renderer
-   **Definition**: A renderer for hierarchical grids (hgridf) that combines primitive generation and rendering in a single pass, updating only 'dirty' cells to support interactive editing [S1, unofficial].
-   **Step**: Traverse the top-level grid and visit each grid-cell unless it is empty [S1, unofficial]. Check if the current grid-cell is 'dirty' (voxels have changed since the last visit) [S1, unofficial]. If dirty, use the associated point or polygon engine to regenerate the local isosurface representation for that cell [S1, unofficial]. Mark the cell as 'clean' after regeneration [S1, unofficial].
-   **Parameter**: The hierarchical grid uses a top-level $N \times N \times N$ grid and sub-grids of size $M \times M \times M$, where $M=16$ [S1, unofficial].

### Multi-resolution grids
-   **Definition**: A recursive hierarchical subdivision algorithm for grids proposed by Jevans and Wyvill [JW89] that integrates regular and adaptive spatial subdivision methods [S2]. The build approach is top-down. This integration allows images consisting of large regularly distributed objects and small dense objects to be ray traced efficiently [S2].
-   **Step**: All primitives are initially inserted into a single voxel. There are two kinds of voxel: leaf voxel, which contains a list of objects inside that voxel, and internal voxel, which maintains a voxel grid subdivision [S2]. Build methods include: octree-like, setting the size of voxel grids to 2 on a side; setting a maximum depth of the subdivision tree; fixing the resolution of the voxel subgrids to a constant value; variable (adaptive) voxel resolution [S2].
-   **Parameter**: Parameters controlling coarseness of the voxel grid, depth of adaptive subdivision trees, and maximum number of polygons per voxel are tailored [S2].
-   **Layout**: A hashtable is used to maintain each non-empty internal voxel [S2].
-   **Pitfall**: Without using a good indexing strategy (such as Jevans and Wyvill hash table), it is not possible to use deep levels of recursion [S2].

## Background
The concept of hierarchical grids originated in the early 1990s as a way to accelerate ray tracing in scenes with varying density. Jevans and Wyvill [JW89] proposed a recursive hierarchical subdivision algorithm for grids, integrating regular and adaptive spatial subdivision methods [S2]. Parker et al. [PPL* 99] introduced the "macrocell" optimization, superimposing a coarser grid over a fine uniform grid to skip empty regions [S2]. The "hgrid" (hierarchical grid) structure, specifically the hgridf renderer, was developed to support interactive editing by combining primitive generation and rendering in a single pass, updating only 'dirty' cells [S1, unofficial]. These structures are widely used in volume rendering and ray tracing of complex scenes, particularly where memory locality and traversal speed are critical [S1, unofficial][S2].

## Sources
[S1] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
[S2] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
