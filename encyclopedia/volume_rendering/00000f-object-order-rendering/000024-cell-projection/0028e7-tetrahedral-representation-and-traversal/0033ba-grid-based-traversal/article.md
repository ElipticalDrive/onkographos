# Grid-based Traversal

Also called: Hierarchical grid traversal for irregular volumes, Parker's brute-force approach, Parker's method

## Overview
Grid-based Traversal is a ray-tracing method for rendering irregular volume data (tetrahedral meshes) at interactive rates by partitioning the volume into a hierarchical grid structure to quickly determine candidate tetrahedra [S1][S2, unofficial]. It sets itself apart from its siblings by avoiding explicit connectivity information, instead relying on spatial hashing into a rectilinear grid to cull non-intersecting cells using min/max value bounds [S1][S2, unofficial].

## How it works
**Inputs:** A tetrahedral mesh representing the volume; a hierarchical rectilinear grid structure; ray origins and directions for each pixel.
**Outputs:** A 2D image with iso-surface values or scalar attributes interpolated at intersection points.

**Data Structures:**
1.  **Hierarchical Rectilinear Grid:** A multi-level grid (e.g., three layers) partitioning 3D space [S1][S2, unofficial].
2.  **Grid Cell:** Contains:
    *   `min_val`: Minimum scalar value of all tetrahedra attached to this cell.
    *   `max_val`: Maximum scalar value of all tetrahedra attached to this cell.
    *   `tetrahedra`: A list of references to all tetrahedra intersecting this cell [S1][S2, unofficial].

**Steps:**
1.  **Grid Construction:**
    *   Partition the volume bounding box into a hierarchical rectilinear grid [S1][S2, unofficial].
    *   For each tetrahedron in the mesh, determine which grid cells it intersects [S1].
    *   Attach the tetrahedron to the list of all intersecting grid cells [S1][S2, unofficial].
    *   For each grid cell, compute and store the minimum (`min_val`) and maximum (`max_val`) scalar values of all attached tetrahedra [S1][S2, unofficial].

2.  **Ray Traversal and Culling:**
    *   For each ray, traverse the hierarchical grid [S1][S2, unofficial].
    *   For each visited grid cell, check if the target iso-value $I$ lies within the cell's bounds:
        $$ \text{min\_val} \le I \le \text{max\_val} $$
    *   If the condition fails, cull the cell and all its attached tetrahedra [S1][S2, unofficial].
    *   If the condition passes, iterate through all tetrahedra attached to the cell [S1].

3.  **Intersection and Interpolation:**
    *   For each candidate tetrahedron, compute the intersection of the ray with the tetrahedron's faces (triangles) [S1].
    *   Determine the barycentric coordinates of the intersection point on the intersected face [S1].
    *   Use the barycentric coordinates to linearly interpolate the scalar value $S$ at the intersection point from the vertex values of the tetrahedron [S1].
    *   Compute the intersection with the iso-surface using linear interpolation of the scalar values along the ray segment within the tetrahedron [S1].
    *   [G] If the interpolated scalar value crosses the iso-value $I$ within the tetrahedron, record the intersection point and depth.

4.  **Rendering:**
    *   The iso-surface is computed on the fly during traversal, allowing dynamic changes to the iso-value [S1].
    *   [G] Standard ray-tracing shading or coloring is applied at the intersection point.

**Parameters:**
*   Grid resolution: Not given in the sources.
*   Number of grid layers: Three layers were used in the reported results [S1].

**Edge Cases and Pitfalls:**
*   **Curvilinear Volumes:** The method does not natively support curvilinear volumes. They must be decomposed into hexahedra, which are then further decomposed into five tetrahedra [S1][S2, unofficial].
*   **Performance:** Since no connectivity information is used, all tetrahedra attached to a leaf cell must be checked, which can be inefficient if many tetrahedra overlap a single cell [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of processors for parallel implementations [S1][S2, unofficial]. The per-ray cost depends on the number of tetrahedra attached to visited grid cells.
*   **Memory Complexity:** Proportional to the number of tetrahedra and grid cells, as each tetrahedron is stored in every cell it intersects [S1][S2, unofficial].
*   **Measured Performance:** Rendered a bioelectric field consisting of over one million tetrahedra at interactive rates using a 512x512 viewport on 16 processors or more [S1][S2, unofficial].
*   **Strengths:** Enables interactive rendering of large irregular volume datasets on supercomputers; allows dynamic iso-value changes [S1].
*   **Limitations:** Does not support curvilinear volumes without preprocessing; potentially high memory usage due to tetrahedron duplication across grid cells [S1][S2, unofficial].

## Background
Grid-based Traversal was introduced by Parker et al. [PPL* 99] as one of the first methods capable of rendering irregular volume data at interactive rates on a supercomputer [S1][S2, unofficial]. It was applied to bioelectric field simulations, demonstrating high-quality volume rendering of large datasets [S1]. The method scales well with the number of processors [S1][S2, unofficial].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Interactive Volume Rendering Techniques, https://www.scribd.com/document/479714183/Interactive-Volume-Rendering-with-Ray-Tracing, fetched 2026-10-25 (unofficial)
