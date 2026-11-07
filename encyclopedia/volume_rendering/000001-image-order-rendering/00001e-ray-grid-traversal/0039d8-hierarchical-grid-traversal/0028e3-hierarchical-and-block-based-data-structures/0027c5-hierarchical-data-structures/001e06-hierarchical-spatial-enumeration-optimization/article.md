# Hierarchical Spatial Enumeration Optimization

Also called: Complete octree representation, Heirarchical Spatial Enumeration, Hierarchical Spatial Enumeration (Octree Pyramid), Pyramid Enumeration, Pyramid of binary volumes

## Overview
Hierarchical Spatial Enumeration Optimization is a ray-tracing acceleration technique for volume rendering that encodes spatial coherence (empty regions) using a pyramid of binary volumes, also known as a complete octree representation [S1][S3, unofficial]. It solves the problem of efficiently skipping large empty regions in volumetric datasets during image-order rendering [S2, unofficial]. Unlike geometric ray tracing which uses bounding volumes or spatial subdivision with lists of primitives, this method uses a hierarchical binary volume structure, which is more efficient for volume data because volume datasets have tens of millions of spatially ordered cells with single samples, whereas geometric scenes have hundreds of cells with many primitives [S1]. It differs from [Jiang et al. Acceleration Structure](1673-jiang-et-al-acceleration-structure.md) by using a pyramid of binary volumes/complete octree rather than a condensed representation [S1].

## How it works
**Inputs:** A volumetric dataset of size $N \times N \times N$ voxels, where $N = 2^M + 1$ for some integer $M$. Each voxel has an associated opacity $\alpha$.
**Outputs:** A pyramid of $M+1$ binary volumes $V_m$ for $m=0, \dots, M$, and the rendered image.

**Data Structures:**
The pyramid consists of $M+1$ volumes $V_m$.
-   **Level $m$ Volume ($V_m$):** A 3D array of binary values.
-   **Size:** $V_m$ has dimensions $\frac{N-1}{2^m} \times \frac{N-1}{2^m} \times \frac{N-1}{2^m}$ cells.
-   **Indexing:** Cells are indexed by level $m$ and vector $\mathbf{i} = (i, j, k)$ where $i, j, k \in \{1, \dots, \frac{N-1}{2^m}\}$.
-   **Value:** $V_m(\mathbf{i}) \in \{0, 1\}$. $0$ indicates an empty region (all constituent voxels have zero opacity), $1$ indicates an occupied region.
-   **Cell Size:** The physical size of a cell on level $m$ is $2^m$ times the spacing between voxels. Voxels are treated as point samples; cells fill the space between them. Each volume is one cell larger in each direction than the underlying dataset [S1][S3, unofficial].
-   **Coordinate Alignment:** Voxel $(1,1,1)$ is placed at the front-lower-right corner of cell $(1,1,1)$ on level $0$. Cell $(1,1,1)$ on level $0$ encloses the space between voxels $(1,1,1)$ and $(2,2,2)$ [S1][S3, unofficial].

**Construction Algorithm:**
The pyramid is constructed bottom-up.

1.  **Base Level ($V_0$):**
    For each cell $\mathbf{i} \in \{1, \dots, N-1\}^3$:
    $$
    V_0(\mathbf{i}) = \begin{cases} 
    1 & \text{if } \alpha(\mathbf{i} + \Delta\mathbf{i}) > 0 \text{ for any } \Delta\mathbf{i} \in \{0, 1\}^3 \\
    0 & \text{otherwise}
    \end{cases}
    $$
    Cell $\mathbf{i}$ in $V_0$ is $1$ if any of the 8 voxels at its vertices has non-zero opacity [S1][S3, unofficial].

2.  **Recursive Levels ($m > 0$):**
    For each cell $\mathbf{i}$ in volume $V_m$:
    $$
    V_m(\mathbf{i}) = \begin{cases} 
    0 & \text{if } V_{m-1}(\mathbf{j}) = 0 \text{ for all 8 children } \mathbf{j} \text{ forming the octant of } \mathbf{i} \\
    1 & \text{otherwise}
    \end{cases}
    $$
    Cell $\mathbf{i}$ in $V_m$ is $0$ if all 8 cells on level $m-1$ that form its octants contain zeros [S1][S3, unofficial].

**Ray Traversal Algorithm:**
For each pixel, a viewing ray is traced from the observer position through the data.

1.  **Entry Point:** Compute the point where the ray enters the single cell at the top level ($V_M$) [S3, unofficial].
2.  **Traversal Loop:**
    Let $\mathbf{c}$ be the current cell index at level $m$.
    -   **Check Occupancy:** If $V_m(\mathbf{c}) == 0$:
        -   Advance the ray to the next cell on the same level $m$ in the direction of the ray [S3, unofficial].
        -   Let $\mathbf{c}_{new}$ be the new cell index.
        -   If the parent of $\mathbf{c}_{new}$ differs from the parent of $\mathbf{c}$, move up to the parent of $\mathbf{c}_{new}$ (increment $m$) [S3, unofficial]. This allows quick advancement across empty regions [S3, unofficial].
        -   Repeat check for the new cell.
    -   If $V_m(\mathbf{c}) == 1$:
        -   If $m > 0$: Move down one level to the enclosing child cell (decrement $m$) [S3, unofficial].
        -   If $m == 0$ (Leaf Level):
            -   The cell contains one or more voxels with opacity $> 0$ [S3, unofficial].
            -   Draw samples at evenly spaced locations along the part of the ray falling within the cell [S3, unofficial].
            -   Resample the data at these sample locations (e.g., using trilinear interpolation) [S1].
            -   Composite the resulting color and opacity into the ray state (front-to-back compositing) [S3, unofficial].
            -   Continue to the next cell on level $0$ [S3, unofficial].

**Parameters:**
-   $M$: Integer such that dataset side length $N = 2^M + 1$. The pyramid has levels $m = 0, \dots, M$ [S1].
-   Opacity Threshold: Voxels with opacity $\alpha = 0$ are considered empty [S2, unofficial].

**Edge Cases and Pitfalls:**
-   **Coordinate Alignment:** Incorrect alignment between voxel centers and cell boundaries can lead to incorrect empty/occupied classification. Voxel $(1,1,1)$ must be at the front-lower-right corner of cell $(1,1,1)$ on level $0$ [S1][S3, unofficial].
-   **Dataset Size:** The method requires $N = 2^M + 1$. Non-conforming sizes require padding or adjustment [S1][S3, unofficial].
-   **Precision:** Ray-cell intersection calculations must be precise to avoid missing small occupied cells or incorrectly skipping occupied regions [S3, unofficial].

## Subtopics

### [Hierarchical Volume Traversal with Adaptive Termination](1682-hierarchical-volume-traversal-with-adaptive-termination.md)
Combines this pyramid structure with an adaptive termination criterion to stop tracing when opacity changes become negligible. Chosen when opacity accumulates quickly or objects are opaque, reducing dependence on depth complexity.

## Operating characteristics
-   **Time Complexity:** Grows linearly with the size of the image, logarithmically with the length of each ray, and linearly with the depth complexity of the data [S2, unofficial].
-   **Memory Complexity:** Requires storage for $M+1$ binary volumes. The total number of cells is $\sum_{m=0}^{M} (\frac{N-1}{2^m})^3$. For large $N$, this is dominated by the base level $V_0$, which has $(N-1)^3$ cells.
-   **Performance:** Speedups of an order of magnitude have been observed for datasets of useful size and complexity [S1]. Saves time by quickly advancing across empty regions of space [S3, unofficial].
-   **Strengths:** Efficiently skips large empty regions; suitable for image-order rendering of dense volumetric data [S1][S3, unofficial].
-   **Limitations:** Requires $N = 2^M + 1$; construction cost is non-trivial; not suitable for geometric scenes with sparse primitives [S1].
-   **When to Use:** Dense volumetric datasets with significant empty regions (e.g., medical imaging, molecular graphics) [S1][S2, unofficial].
-   **When Not to Use:** Geometric scenes with few primitives; datasets where $N$ cannot be easily adjusted to $2^M + 1$ without excessive padding [S1].

## Background
Originated in the late 1980s by Marc Levoy. The technique was introduced in "Efficient ray tracing of volume data" (1988) [S1] and further detailed in "Rendering Mixtures of Geometric and Volumetric Data" (1989) [S2, unofficial]. It was developed to address the high computational cost of ray casting for large volumetric datasets by exploiting spatial coherence [S1][S2, unofficial]. It is used in medical imaging and molecular graphics applications [S1].

## Sources
[S1] Efficient ray tracing of volume data, https://doi.org/10.1145/78964.78965, fetched 2026-10-25
[S2] DTIC ADA208085: Rendering Mixtures of Geometric and Volumetric Data, https://archive.org/details/DTIC_ADA208085, fetched 2026-10-26 (unofficial)
[S3] Volume Rendering With Ray Casting - WPI, https://web.cs.wpi.edu/~matt/courses/cs563/talks/powwie/p1/ray-cast.htm, fetched 2026-10-26 (unofficial)
