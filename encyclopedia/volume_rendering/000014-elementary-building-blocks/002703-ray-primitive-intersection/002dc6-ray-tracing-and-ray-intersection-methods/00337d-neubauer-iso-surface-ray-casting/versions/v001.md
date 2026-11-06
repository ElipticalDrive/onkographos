# Neubauer Iso-surface Ray Casting
Also called: Kd-Jump, Neubauer method

## Overview
Neubauer Iso-surface Ray Casting is a single-kernel rendering technique that accelerates iso-surface extraction by combining spatial subdivision with projection-based culling [S1, unofficial]. It differs from sibling methods like Binary Search IsoSurface Extraction by avoiding iterative root-finding in favor of geometric footprint culling, and from Trilinear Isosurface Intersection by using a coarse octree hierarchy to skip empty regions rather than sampling the volume directly [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A 3D scalar volume dataset, an iso-value threshold, and camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$, image plane resolution).
*   **Output:** A 2D image where pixels are colored based on the intersection of rays with the iso-surface.

**Data Structures**
1.  **Macro-cells:** The volume is subdivided into cubic cells of size $m^3$.
2.  **Min/Max Octree:** A hierarchical structure built from the macro-cells. Each node stores the minimum and maximum scalar values found within its spatial extent [S1, unofficial]. This allows for early termination during traversal if the iso-value lies outside the range $[min, max]$ of a node [S1, unofficial].

**Parameters**
*   $m$: The dimension of the macro-cell side length. Typical values are between 4 and 10 [S1, unofficial].

**Steps**
1.  **Preprocessing:**
    1.  Subdivide the complete dataset into macro-cells of size $m^3$ [S1, unofficial].
    2.  Compute the min/max scalar values for each macro-cell.
    3.  Build the min/max octree from these macro-cells [S1, unofficial].

2.  **Ray Traversal (Per Pixel):**
    1.  Initialize the ray with origin $\mathbf{o}$ and direction $\mathbf{d}$ [Parent].
    2.  Traverse the octree from the root.
    3.  At each internal node, check if the iso-value lies within the node's $[min, max]$ range. If not, prune the subtree [S1, unofficial].
    4.  If the node is a leaf (macro-cell):
        1.  Project the boundaries of the macro-cell onto the image plane to determine the set of pixels covered by this cell. This projection yields a hexagonal footprint [S1, unofficial].
        2.  For each pixel in this footprint, generate a local ray corresponding to that pixel's position within the macro-cell [S1, unofficial].
        3.  Traverse the macro-cell grid (the $m^3$ subdivision) using these local rays to find the iso-surface intersection [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Precision:** The projection of 3D cell boundaries to 2D image space requires accurate perspective or orthographic projection matrices.
*   **Boundaries:** The hexagonal footprint calculation must correctly handle cells at the volume boundaries to avoid out-of-bounds memory access [S1, unofficial].

## Subtopics

## Operating characteristics
*   **Performance:** Reported average frame rate of 0.8 fps with an image resolution of $512 \times 512$ pixels on a single Pentium IV 1.9 GHz processor [S1, unofficial].
*   **Complexity:** The octree traversal reduces the number of steps for pixels that do not intersect the volume or intersect empty regions. The hexagonal footprint culling ensures that only relevant pixels are processed for each macro-cell, reducing redundant ray generation [S1, unofficial].
*   **Strengths:** Single-kernel implementation suitable for GPU execution; efficient culling of empty space via the min/max octree.
*   **Limitations:** Performance is limited by the coarse nature of the macro-cells and the overhead of projecting boundaries to the image plane. The reported performance on older hardware suggests it may be less efficient than modern GPU-accelerated ray casters [S1, unofficial].

## Background
The method was introduced by Neubauer et al. [NMHW02] as an approach for interactive volume rendering. It is cited in later works as a baseline for comparison, specifically in the context of single-kernel rendering pipelines [S2, unofficial]. The technique relies on the concept of min/max octrees for spatial acceleration, similar to methods described by Westermann et al. [WV92] and [WFM*05] [S1, unofficial].

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S2] InKernel Stream Compaction and Its Application to ..., https://www.sci.utah.edu/~knolla/hughes13.pdf, fetched 2026-10-29 (unofficial)
