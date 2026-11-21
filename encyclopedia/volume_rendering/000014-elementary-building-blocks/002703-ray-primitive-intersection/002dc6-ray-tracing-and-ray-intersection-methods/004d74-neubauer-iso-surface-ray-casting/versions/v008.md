# Neubauer Iso-surface Ray Casting
Also called: Kd-Jump, Neubauer method, Ray Casting Visibility Test, Ray tracing for occlusion

## Overview
Neubauer Iso-surface Ray Casting is a visibility testing method used to determine which parts of extracted line loops or sparse surface features are occluded by the isosurface, avoiding the computational cost of extracting and rendering the entire isosurface mesh [S2]. It sets itself apart from siblings like [Binary Search IsoSurface Extraction](1316-binary-search-isosurface-extraction.md) and [Trilinear Isosurface Intersection](2213-trilinear-isosurface-intersection.md) by focusing on binary occlusion checks for specific vertices rather than precise surface point extraction, and differs from [Ray Tracing](8334-ray-tracing.md) by using a coarse macro-cell traversal structure optimized for sparse feature visibility [S1, unofficial][S2].

## How it works
**Inputs:**
*   A scalar volume dataset $V$.
*   A set of vertices $V_{verts}$ representing features (e.g., line loop vertices) to be tested for visibility [S2].
*   Camera position $C$ [S2].
*   Macro-cell size parameter $m$ [S1, unofficial].

**Outputs:**
*   A boolean mask indicating which vertices in $V_{verts}$ are occluded by the isosurface [S2].

**Data Structures:**
1.  **Macro-Cell Grid:** The volume is subdivided into a 3D grid of macro-cells, each of size $m \times m \times m$ voxels [S1, unofficial].
2.  **Min/Max Octree:** Built from the macro-cells. Each leaf node stores the minimum and maximum scalar values found within its corresponding macro-cell [S1, unofficial]. This structure allows for early rejection of empty or non-intersecting regions [S1, unofficial].

**Steps:**
1.  **Preprocessing:**
    1.  Subdivide the complete data set into macro-cells of size $m^3$ [S1, unofficial].
    2.  Build a min/max octree from these macro-cells. For each macro-cell, compute $min\_val = \min(V)$ and $max\_val = \max(V)$ within that cell [S1, unofficial].
    3.  Set the macro-cell size parameter $m$, typically between 4 and 10 [S1, unofficial].

2.  **Visibility Test for Each Vertex:**
    For each vertex $v$ in the rendered feature set:
    1.  Define a ray $R(t) = v + t(C - v)$, where $t > 0$, originating at $v$ and directed towards the camera $C$ [S2].
    2.  Apply a depth offset heuristic to the ray origin to prevent immediate self-intersection detection [S2].
    3.  Traverse the min/max octree using the ray:
        a.  At each node, check if the ray intersects the node's bounding volume.
        b.  If the node is an internal node, check if the ray could intersect the isosurface by comparing the ray's scalar values against the node's stored min/max values. If the isosurface value $iso$ is not within $[min\_val, max\_val]$, prune the branch [S1, unofficial].
        c.  If the node is a leaf (macro-cell):
            i.  Project/rasterize the boundaries of the macro-cell onto the image plane to yield a hexagonal footprint [S1, unofficial].
            ii. For the specific ray, determine the exit face of the current macro-cell [S2].
            iii. Calculate the intersection point $P$ of the ray with this exit face [S2].
            iv. Perform linear interpolation of the scalar field at $P$ using the four corners of the face (or appropriate voxel corners defining the face geometry) to determine the function value $f(P)$ [S2].
            v.  Check for a sign change. If the ray passes from outside to inside the isosurface (i.e., $f(P)$ indicates crossing the threshold $iso$), mark the originating vertex $v$ as occluded and terminate the ray trace for this vertex [S2].

**Parameters:**
*   $m$: Macro-cell size dimension. Typical values are between 4 and 10 [S1, unofficial].
*   $iso$: The isosurface threshold value [S2].

**Edge Cases and Pitfalls:**
*   **Self-Intersection:** Tracing a ray from a vertex on the surface directly to the camera may immediately detect the surface itself as an occluder. This is mitigated by using a "depth offset" heuristic, moving the ray origin slightly away from the vertex [S2].
*   **Precision:** Linear interpolation at the cell boundary assumes the scalar field is well-behaved within the macro-cell. If the isosurface is highly complex within a single macro-cell, the min/max octree pruning may fail to reject branches that actually contain intersections, leading to false negatives (non-occluded vertices marked as visible) or unnecessary traversal [S1, unofficial][S2].

## Subtopics

### [Bézier Clipping for Metaball Ray Casting](9481-b-zier-clipping-for-metaball-ray-casting.md)
This method computes the ray-isosurface intersection for metaballs by representing the scalar density field along a ray as a polynomial in Bézier form and applying Bézier clipping to find the root. It sets itself apart from standard Newton-Raphson approaches by providing robust root isolation and handling multiple intersections within a single interval without derivative evaluation, building on [Neubauer Iso-surface Ray Casting](7024-neubauer-iso-surface-ray-casting.md).

### Interactive Large Iso-Surface Ray Tracing
A distributed implementation of the Neubauer method optimized for large datasets using a cluster of render nodes. It sets itself apart by scaling performance linearly with the number of nodes, addressing the scalability limitations of the single-node Neubauer approach. It is chosen when rendering very large datasets (e.g., >400 MB) where single-node performance is insufficient. Performance is almost five times slower than Wald's kd-tree approach for the Visible Female dataset on a single PC, but scales effectively on clusters [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Asymptotically $O(n^2)$ for a volume of size $n^3$. The number of vertices rendered is $O(n)$, and the number of cells touched by an average ray is also $O(n)$ [S2].
*   **Memory Complexity:** Requires storage for the min/max octree. The memory overhead is significantly less than other acceleration structures like kd-trees for large datasets [S1, unofficial].
*   **Measured Performance:**
    *   Single PIV 1.9 GHz, 512x512 resolution: Average frame rate of 0.8 fps [S1, unofficial].
    *   Visible Female dataset (428 MB): 1 fps with 1 render node, up to 22 fps with 32 nodes [S1, unofficial].
    *   Compared to Wald's approach (~8.1 fps on single PC), Neubauer is ~5x slower on the Visible Female dataset on a single machine [S1, unofficial].
*   **Strengths:** Avoids the cost of full isosurface extraction; suitable for sparse features; low memory overhead for acceleration structures [S1, unofficial][S2].
*   **Limitations:** Does not scale well for real-time rendering of large datasets on single nodes; asymptotically expensive compared to z-buffer approaches for dense features; precision limited by macro-cell size and linear interpolation [S1, unofficial][S2].
*   **When to use:** When only sparse features (lines, loops) need visibility testing and full mesh extraction is too costly.
*   **When not to use:** When real-time performance on large, dense volumes is required on a single CPU; when precise surface point extraction is needed rather than binary occlusion [S1, unofficial][S2].

## Background
The method was introduced by Neubauer et al. in 2002 [S1, unofficial][S2]. It was developed as a solution for rendering line drawings from volume data, specifically to determine occlusion for extracted loops without generating the full isosurface mesh [S2]. It has been used as a baseline for comparison in single-kernel rendering pipelines, such as in the Kd-Jump implementation [S3, unofficial].

## Sources
[S1] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
[S2] Line drawings from volume data, https://doi.org/10.1145/1073204.1073222, fetched 2026-10-28
[S3] InKernel Stream Compaction and Its Application to ..., https://www.sci.utah.edu/~knolla/hughes13.pdf, fetched 2026-10-29 (unofficial)
