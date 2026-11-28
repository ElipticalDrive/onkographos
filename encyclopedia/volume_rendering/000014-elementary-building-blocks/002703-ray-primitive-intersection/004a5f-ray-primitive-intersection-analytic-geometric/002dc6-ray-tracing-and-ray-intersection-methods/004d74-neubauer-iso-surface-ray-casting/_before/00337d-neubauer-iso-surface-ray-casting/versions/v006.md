# Neubauer Iso-surface Ray Casting

Also called: Kd-Jump, Neubauer method

## Overview
Neubauer Iso-surface Ray Casting is a single-kernel iso-surface ray casting method that uses a min/max octree built from macro-cells to accelerate ray traversal [S2, unofficial]. It sets itself apart from sibling methods like Binary Search IsoSurface Extraction by using spatial subdivision and projection-based rasterization to skip empty space, rather than relying solely on iterative root-finding within dense grids [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A scalar volume dataset, an iso-value threshold, and a camera model defining rays.
*   **Output:** A rendered image where pixels intersecting the iso-surface are colored, and empty space is transparent or background-colored.

**Data Structures**
1.  **Macro-cells:** The complete data set is subdivided into cubic macro-cells of size $m^3$.
2.  **Min/Max Octree:** A hierarchical structure built from these macro-cells. Each node stores the minimum and maximum scalar values found within its bounding volume. This allows for early termination during traversal if the iso-value lies outside the $[min, max]$ range of a node [S1, unofficial].

**Parameters**
*   $m$: Macro-cell size dimension. Typical values are between 4 and 10 [S1, unofficial].

**Steps**
1.  **Preprocessing:**
    1.  Subdivide the volume into macro-cells of size $m^3$.
    2.  Compute the min and max scalar values for each macro-cell.
    3.  Build the min/max octree from these cells [S1, unofficial].
2.  **Ray Traversal (Per Pixel):**
    1.  For each pixel, cast a ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$.
    2.  Traverse the octree from the root.
    3.  At each node, check if the iso-value lies within the node's $[min, max]$ range. If not, prune the subtree (skip the cell) [S1, unofficial].
    4.  If the node is a leaf (a macro-cell):
        a.  Project the boundaries of the macro-cell onto the image plane.
        b.  This projection yields a hexagonal footprint on the image plane [S1, unofficial].
        c.  Identify all pixels covered by this hexagon.
        d.  For these pixels, perform local ray traversal within the macro-cell grid to find the intersection with the iso-surface [S1, unofficial].
3.  **Intersection:**
    *   Within the macro-cell, standard ray-marching or intersection techniques are used to find the specific point where the scalar field equals the iso-value [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** The projection of 3D cell boundaries to 2D screen space must be handled carefully to avoid gaps or overlaps in the hexagonal footprints, which could lead to missed intersections or redundant calculations [S1, unofficial].
*   **Boundary Conditions:** Rays that do not intersect any macro-cell containing the iso-surface should be terminated early to save computation [S1, unofficial].

## Subtopics

### Interactive Large Iso-Surface Ray Tracing
This variant extends the Neubauer method to cluster-based rendering for large datasets. It sets itself apart by distributing the octree traversal and rendering across multiple nodes. It is chosen when rendering very large datasets (e.g., >400 MB) that exceed single-node memory or processing capabilities. Performance scales linearly with the number of render nodes [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The octree traversal reduces the number of steps compared to linear voxel traversal, but the complexity depends on the density of the iso-surface and the macro-cell size $m$.
*   **Memory Complexity:** The min/max octree requires additional memory proportional to the number of macro-cells. It consumes less additional memory than some other acceleration structures like those used in DeMarle's approach [S1, unofficial].
*   **Measured Performance:**
    *   On a single Pentium IV 1.9 GHz processor with 512x512 resolution, the average frame rate is 0.8 fps [S1, unofficial].
    *   In the Interactive Large Iso-Surface Ray Tracing variant, the Visible Female dataset (428 MB) achieves 1 fps with 1 render node and up to 22 fps with 32 nodes [S1, unofficial].
*   **Strengths:** Efficient space-leaping via the min/max octree; reduced memory overhead compared to some alternatives; scalable to clusters [S1, unofficial].
*   **Limitations:** Slower than Wald's kd-tree approach for certain datasets (e.g., Visible Female: ~1-22 fps on cluster vs. ~8.1 fps on single PC for Wald) [S1, unofficial]. The projection-based rasterization adds complexity to the pixel coverage calculation [S1, unofficial].
*   **When to use:** Suitable for single-kernel implementations where memory is constrained, or for cluster-based rendering of large datasets.
*   **When not to use:** When maximum single-node performance is required and a kd-tree based approach (like Wald's) is feasible [S1, unofficial].

## Background
The method was introduced by Neubauer et al. in 2002 [S1, unofficial]. It is cited as a baseline single-kernel iso-surface ray caster in comparative studies [S2, unofficial]. The approach builds on earlier work in min/max octrees for volume rendering [WV92, WFM*05] and projection-based ray casting [WMKE04] [S1, unofficial].

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S2] InKernel Stream Compaction and Its Application to ..., https://www.sci.utah.edu/~knolla/hughes13.pdf, fetched 2026-10-29 (unofficial)
