# Sorting and Visibility Ordering

Also called: Visibility Ordering

## Overview
Sorting and Visibility Ordering resolves the requirement for back-to-front processing in cell projection volume rendering, ensuring that opacity accumulation follows the correct depth order to prevent visual artifacts [S1, unofficial]. It differs from its siblings by offering a spectrum of solutions ranging from exact topological guarantees for convex meshes ([Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md)) to parallelizable block-based peeling ([Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)) and computationally cheaper approximations ([Approximate Depth Sorting](2016-approximate-depth-sorting.md)).

## Shared principles
The sorting methods in this family share the following mechanism, terms, and math:

1.  **Back-to-Front Requirement:** Correct volume compositing requires fragments or cells to be processed in back-to-front order relative to the viewer [S1, unofficial].
2.  **Occlusion Logic:** The ordering must account for geometric occlusion where one cell blocks the view of another, rather than relying solely on scalar depth values which can be misleading [S1, unofficial].
3.  **Viewpoint Dependency:** The visibility order is generally view-dependent; for accurate methods, the sort must be recomputed when the viewpoint changes .

## Subtopics

### [Topological Sort for Cell Projection](999-topological-sort-for-cell-projection.md)
An object-space algorithm that computes a correct back-to-front visibility ordering of convex polyhedral cells by constructing a directed adjacency graph based on face-sharing topology and viewpoint orientation. It relies on local ordering relations between cells sharing a common face to define a partial order, which a topological sort converts to a total order. Choose this when exact correctness is required for convex meshes and performance allows for $O(N)$ graph traversal.

### [Parallelized Accurate Sorting for Projected Tetrahedra](1128-parallelized-accurate-sorting-for-projected-tetrahedra.md)
A view-dependent sorting algorithm designed to resolve the parallelization bottlenecks inherent in standard Projected Tetrahedra rendering. It uses logical partitioning to divide the dataset into blocks along the viewing direction, enabling the parallel peeling of unobstructed layers. Choose this for large-scale parallel rendering on GPUs where topological dependencies prevent naive parallelization, but exact geometric accuracy is still required.

### [Approximate Depth Sorting](2016-approximate-depth-sorting.md)
A sorting scheme that reduces ray-segment list length and manipulation cost by ordering cells according to their center of gravity depth. It differs from exact methods by being simpler and less costly, accepting that perfect depth order is impossible due to pixel-by-pixel variations, while introducing no quality error in the final image. Choose this for performance-critical applications where the overhead of exact sorting is prohibitive and the dataset structure allows for centroid-based approximation.

## Background
The necessity for visibility ordering in cell projection stems from the front-to-back or back-to-front compositing equations used in volume rendering [S1, unofficial]. Early implementations highlighted that simple centroid-based or power-distance sorts can fail from certain viewpoints, leading to incorrect occlusion handling [S1, unofficial]. The Topological Sort for Cell Projection (MPVO) method was developed to provide a rigorous solution for convex meshes by leveraging the mesh's adjacency graph [S1, unofficial]. Subsequent developments focused on parallelizing this process for modern GPU architectures, leading to block-based peeling techniques [S1, unofficial]. Approximate methods were also explored to mitigate the high cost of sorting large numbers of cells, particularly in structured data contexts .

## Sources
[S1] A Survey of GPU-Based Volume Rendering of ..., https://www.sci.utah.edu/~csilva/papers/rita2005.pdf, fetched 2026-10-29 (unofficial)
