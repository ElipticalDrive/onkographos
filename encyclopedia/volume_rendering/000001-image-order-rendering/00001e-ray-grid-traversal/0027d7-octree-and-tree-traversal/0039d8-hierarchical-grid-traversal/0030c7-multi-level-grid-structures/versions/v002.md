# Multi-level Grid Structures


## Overview
Multi-level Grid Structures are a family of hierarchical acceleration techniques for ray-grid traversal that organize scene data into a hierarchy of grid levels to skip empty space and reduce intersection tests [G]. This family builds on [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md) by providing specific implementations for constructing and traversing these hierarchical grids [G].

## Shared principles
The family shares a hierarchical decomposition of the scene bounding box into a coarse grid, where occupied cells may contain finer sub-grids, and empty cells are skipped during traversal [G].

## Subtopics

### [Two-Level Grids for Ray Tracing on GPUs](4245-two-level-grids-for-ray-tracing-on-gpus.md)
A fixed-depth hierarchical acceleration structure designed to handle non-uniform primitive distributions in dynamic scenes more robustly than uniform grids. It uses a sparse top-level grid where each cell contains a sub-grid (leaf grid) of arbitrary resolution, allowing variable density handling. Unlike uniform grids which suffer from poor acceleration in sparse regions, this method leverages coherence within top-level cells to offer a SIMD-friendly approach that minimizes code divergence, though it offers inferior traversal performance compared to state-of-the-art static scene structures like the Aila-Laine grid.

### [Macrocell Grid Traversal](5207-macrocell-grid-traversal.md)
A hierarchical acceleration technique for ray-grid traversal that superimposes a coarse grid over a base uniform fine grid to skip empty regions of the scene. It solves the problem of redundant traversal steps through empty space in large, sparse volume grids by grouping fine grid cells into blocks. Unlike its sibling [Two-Level Grids for Ray Tracing on GPUs](4245-two-level-grids-for-ray-tracing-on-gpus.md), which may use more complex hierarchical structures or different memory layouts, Macrocell Grid Traversal uses a simple boolean flag per block to indicate occupancy, allowing for trivial build times and straightforward traversal logic.

### Multi-Level Array Grid
#### definition
A classic multi-level grid implementation where each cell is recursively refined according to the number of items it contains, used for performance comparison [S1, unofficial].

#### step
1. Compute the scene's bounding box. Initialize the first node as a leaf node containing all scene items [S1, unofficial].
2. Recursively process each node: if the number of items in the node is less than 8 or the grid depth is over 2, the node remains a leaf and recursion stops [S1, unofficial].
3. Compute delta and allocate the cells array with size N [S1, unofficial].
4. Compute item lists for each cell of the inner node. If the item list is empty, mark the cell pointer as null. If not empty, initialize the cell as a leaf node containing the items. Recurse [S1, unofficial].

#### parameter
- Grid depth limit: recursion stops if depth is over 2 [S1, unofficial].
- Leaf threshold: nodes with fewer than 8 items remain leaves [S1, unofficial].
- Density factor (rho): typically 4 [S1, unofficial].

#### layout
- InnerNode struct: `float min[3]; float max[3]; int dim[3]; float delta[3]; GridNode** cells;` [S1, unofficial].
- GridNode struct: `int index;` (low bit: inner (1) or leaf (0)); `union { struct { int* items; } leaf; struct { InnerNode* interior; } inner; }` [S1, unofficial].

### Hierarchical Grid (hgridf) Renderer
#### definition
A renderer for hierarchical grids (hgridf) that combines primitive generation and rendering in a single pass, updating only 'dirty' cells to support interactive editing [S2, unofficial].

#### step
1. Traverse the top-level grid and visit each grid-cell unless it is empty [S2, unofficial].
2. Check if the current grid-cell is 'dirty' (voxels have changed since the last visit) [S2, unofficial].
3. If dirty, use the associated point or polygon engine to regenerate the local isosurface representation for that cell [S2, unofficial].
4. Mark the cell as 'clean' after regeneration [S2, unofficial].

#### parameter
- The hierarchical grid uses a top-level N×N×N grid and sub-grids of size M×M×M, where M=16 [S2, unofficial].

## Background
The concept of multi-level grids originated in the context of ray tracing acceleration structures, with early implementations focusing on array-based hierarchical grids for performance comparison against single-level grids [S1, unofficial]. The hierarchical grid (hgrid) was introduced to save space in synthetic volumes representing solids of homogeneous material, using a simple two-level structure where each top-level cell is either empty or an M×M×M sub-grid [S2, unofficial]. These structures are used in volume graphics and point rendering applications [S2, unofficial].

## Sources
[S1] SCITEPRESS - SCIENCE AND TECHNOLOGY PUBLICATIONS, https://www.scitepress.org/PublishedPapers/2010/28153/, fetched 2026-10-24 (unofficial)
[S2] Hardware Accelerated Point Rendering of Isosurfaces, http://wscg.zcu.cz/wscg2003/Papers_2003/J61.pdf, fetched 2026-10-26 (unofficial)
