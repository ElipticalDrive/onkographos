# Zalik et al. Algorithm
Also called: ZCO97, Zalik et al. code-based traversal
## Overview
The Zalik et al. algorithm is a voxel traversal method that corrects the topological incompleteness of standard 3D Bresenham tracing by ensuring all cells intersected by a ray are visited [S1][S2, unofficial]. It differs from its parent [3D Bresenham Voxel Tracing](3949-3d-bresenham-voxel-tracing.md) by adding a post-processing phase to recover missed cells, whereas siblings like [Integer-based Amanatides and Woo Voxel Traversal](4246-integer-based-amanatides-and-woo-voxel-traversal.md) compute intersections analytically [S1].
## How it works
**Inputs:** Ray origin $O$, ray direction $D$, grid dimensions, and the driving axis (e.g., $x$) [S1].
**Outputs:** A sequence of voxel coordinates $(x, y, z)$ representing all cells intersected by the ray [S1].

**Data Structures:**
- `current_cell`: Tuple $(x, y, z)$ of integers.
- `next_cell`: Tuple $(x', y', z')$ of integers.
- `error`: Floating-point value tracking the deviation of the ray from the integer grid line [S1].

**Steps:**
1. **Bresenham Phase:** Execute standard 3D Bresenham algorithm along the driving axis to generate a primary sequence of cells [S1]. At each step, the algorithm selects one adjacent cell (e.g., between $(x+1, y)$ and $(x+1, y+1)$) based on the error term $e$ compared against a threshold (typically $1/2$) [S1].
2. **Relationship Examination Phase:** For every pair of successive cells $c_i$ and $c_{i+1}$ in the Bresenham sequence, examine their geometric relationship [S1].
3. **Recovery:** If the ray passes through a corner or edge shared by multiple voxels, the standard Bresenham step may skip one or more valid cells [S1]. The algorithm identifies these missing cells by analyzing the coordinate differences between $c_i$ and $c_{i+1}$ and inserts them into the traversal sequence [S1].
4. **Iteration:** Repeat steps 2-3 until the ray exits the grid [S1].

**Pitfalls:**
- The algorithm requires elaborate implementation logic to determine which cells are "missed" based on the relationship between successive cells [S1].
- Higher traversal time compared to simpler methods due to the additional examination phase [S1].

## Operating characteristics
- **Time Complexity:** Higher than standard Bresenham due to the second phase of examining relationships between successive cells [S1].
- **Memory Complexity:** Low, storing only current and next voxel coordinates [S1].
- **Strengths:** Guarantees topological correctness by visiting all intersected cells, unlike standard Bresenham [S1][S2, unofficial].
- **Limitations:** Elaborate implementation and increased computational overhead per step [S1].
- **When to use:** When strict topological correctness is required and performance is secondary to completeness [S1].
- **When not to use:** When performance is critical and minor topological errors are acceptable, or when using more efficient extensions like the Liu Extension [S1].

## Background
The algorithm was proposed by Zalik et al. in 1997 (ZCO97) to address the limitation of Bresenham's algorithm, which, when a driving axis is chosen, selects only one cell between adjacent options (e.g., $(x+1, y)$ and $(x+1, y+1)$) and never both, thus missing cells traversed by the ray [S1][S2, unofficial]. It is part of the family of code-based traversal algorithms [S1].

## Sources
[S1] A Survey on Exploiting Grids for Ray Tracing, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/089-096, fetched 2026-10-24
[S2] Grids in Ray Tracing: A Comprehensive Survey, https://www.scribd.com/document/349616186/GG-400, fetched 2026-10-25 (unofficial)
