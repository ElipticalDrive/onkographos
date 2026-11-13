# Projected Tetrahedra Rendering

Also called: PT Rendering

## Overview
Projected Tetrahedra Rendering is an object-order volume rendering technique that decomposes a volume into tetrahedral cells, projects them to screen space, and composites the resulting fragments back-to-front. It solves the topological ambiguities of curvilinear grids and the connectivity overhead of large unstructured meshes by leveraging the planar faces of tetrahedra [S1, S2, unofficial][S4]. It sets itself apart from ray-casting methods by avoiding per-ray traversal and from standard polygonal rendering by handling semi-transparent volumes with depth-aware compositing rather than simple opaque blending.

## Shared principles
The cell projection family shares the following mechanism, terms, and math:

1.  **Primitive Representation:** The volume is decomposed into convex polyhedral cells (e.g., tetrahedra, hexahedra). For regular grids (cubes or hexahedra), cells are often subdivided into tetrahedra (typically 5 or 6) to ensure planar faces and consistent topology [S2, S8, S13, S19, unofficial][S1, unofficial]. This decomposition resolves "problem cells" where rays might intersect a face twice in curvilinear grids [S4].
2.  **Projection and Subdivision:** The 3D vertices of each cell are projected to 2D screen space. The projected edges of the cell divide the image plane into polygons, where each polygon lies within the projection of a single front-facing face and a single back-facing face of the cell [S1, S2, unofficial][S4]. For general cells, an incremental method builds this subdivision by adding projected edges one by one [S1, S2, unofficial]. For curvilinear grids, standard topologies are identified to use lookup tables for subdivision [S1, S2, unofficial].
3.  **Scalar Interpolation:** Scalar values $s$ at any point within a tetrahedron are interpolated linearly from the vertex values $s_i$ using barycentric coordinates or linear interpolation along edges [S1, unofficial][S2, unofficial][S3]. The scalar field is assumed to vary linearly across the cell, and transfer functions mapping scalar to color/extinction are assumed linear [S1, S2, unofficial][S4].
4.  **Opacity Calculation:** The integrated opacity $\alpha$ for a segment of length $l$ with average differential opacity $\tau_{avg}$ is:
    $$ \alpha = 1 - \exp(-\tau_{avg} l) $$
    [S1, unofficial][S4].
5.  **Visibility and Sorting:** Correct compositing requires back-to-front ordering. This is achieved via global visibility sorts (e.g., sorting by centroid depth) or image-space merging of ray segments [S1, unofficial][S4, S7, S11][S12, unofficial][S14][S17, unofficial].
6.  **Compositing:** Fragments or ray segments are composited using the Porter-Duff over operation in back-to-front order [S1, unofficial][S2, unofficial][S6, S7][S9, S10, unofficial][S11][S12, unofficial][S14, S16][S17, S18, unofficial].

## Subtopics

### [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)
A view-dependent sorting algorithm using logical partitioning to enable parallel processing, solving dependency limitations of neighboring tetrahedra. Choose when parallel performance on large datasets is critical.

### [Point-based Projected Tetrahedra](5527-point-based-projected-tetrahedra.md)
Approximates each tetrahedron with a single point primitive, decoupling footprint generation from compositing. Choose for interactive visualization of large, unstructured meshes where robustness to malformed tetrahedra is needed.

### [Sorting-free Pre-integrated Projected Tetrahedra](7491-sorting-free-pre-integrated-projected-tetrahedra.md)
A stochastic method treating volume rendering as a particle process, eliminating the need for visibility sorting. Choose when sorting overhead is prohibitive and stochastic noise is acceptable.

## Background
Projected Tetrahedra rendering originated as a solution for accurate volume visualization of unstructured grids, addressing the limitations of ray-casting in terms of speed and the topological issues of curvilinear grids [S1, S2, unofficial][S4]. It builds on the principles of cell projection and tetrahedral decomposition [S1, S2, unofficial].

## Sources
[S1] Parallelized Accurate Sorting for Projected Tetrahedra, https://example.com/1128, 2023-10-01
[S2] Point-based Projected Tetrahedra, https://example.com/5527, 2023-10-01
[S3] Sorting-free Pre-integrated Projected Tetrahedra, https://example.com/7491, 2023-10-01
[S4] Tetrahedral representation and traversal, https://example.com/4396, 2023-10-01
[S6] Projected Tetrahedra, https://example.com/877, 2023-10-01
[S7] Marching Tetrahedrons Algorithm, https://example.com/940, 2023-10-01
[S8] Early Tetrahedron Termination, https://example.com/1451, 2023-10-01
[S9] Adaptive Delaunay Update, https://example.com/4923, 2023-10-01
[S10] Grid-based Traversal, https://example.com/7083, 2023-10-01
[S11] Parallel Projected Tetrahedra with KD-Tree Partitioning, https://example.com/8767, 2023-10-01
[S12] Tetrahedral Mesh Connectivity Encoding, https://example.com/8957, 2023-10-01
[S13] Delaunay Triangulation for Tetrahedral Representation, https://example.com/9905, 2023-10-01
[S14] TetraSDF, https://example.com/11632, 2023-10-01
[S16] Tetrahedral Primitive Representation, https://example.com/12385, 2023-10-01
[S17] Projected Tetrahedra Rendering, https://example.com/pt-family, 2023-10-01
[S18] Volume Rendering Principles, https://example.com/vr-principles, 2023-10-01
[S19] Unstructured Grid Visualization, https://example.com/unstructured, 2023-10-01
