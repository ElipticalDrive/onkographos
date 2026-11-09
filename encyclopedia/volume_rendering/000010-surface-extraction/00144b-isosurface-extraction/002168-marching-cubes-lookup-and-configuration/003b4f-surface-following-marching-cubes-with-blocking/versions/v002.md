# Surface-following Marching Cubes with Blocking


## Overview
Surface-following Marching Cubes with Blocking is a surface extraction method designed for unbounded rendering of particle-based simulations [S1]. It addresses the performance bottlenecks of standard voxel-based Marching Cubes by treating the isosurface as a connected manifold grown from seed triangles rather than processing every voxel in a dense grid [S1]. Unlike standard MC, which evaluates each cube independently and may produce inconsistent topology at shared edges, this method propagates the surface through adjacent cubes via a queue of "growth edges," ensuring topological consistency across the entire reconstructed model [S1]. It is particularly suited for 3D reconstruction where only the main contour is required or for isolating specific connected components from scattered data by controlling the seed region [S1].

## How it works
The method combines a surface-following traversal strategy with a blocking algorithm and a specialized 3D lookup cache for particle queries [S1].

**Inputs and Outputs:**
*   **Input:** A set of particles with positions and scalar values; an isovalue $I$; a seed region or initial triangle set.
*   **Output:** A manifold triangle mesh representing the isosurface.

**Data Structures:**
1.  **Growth Queue:** A queue storing edges or faces that need to be processed to extend the surface.
2.  **Visited Set:** A hash map or set storing processed voxels or edges to prevent re-processing and cycles.
3.  **3D Lookup Cache:** A hash map used for particle lookup. Unlike separate coarse grids used in other approaches, this cache allows for exact lookups within a radius $R_c$ without returning particles outside the search volume [S1].

**Algorithm Steps:**
1.  **Initialization:** Identify seed triangles or voxels where the isosurface intersects. Add the boundary edges of these initial triangles to the Growth Queue. Mark these edges as visited.
2.  **Surface Following:** While the Growth Queue is not empty:
    a.  Dequeue an edge $e$.
    b.  Identify the voxel(s) adjacent to $e$ that have not been fully processed.
    c.  Perform a particle lookup within the voxel using the 3D lookup cache to find particles within radius $R_c$ of the voxel center or relevant region [S1]. The cache is designed to return only particles within the exact spherical volume $(4/3)\pi R_c^3$, avoiding the false positives inherent in cubic grid lookups of size $3 \times 3 \times 3$ [S1].
    d.  Evaluate the scalar field at the voxel corners using the particle data.
    e.  Compute the case index $C$ based on the sign of $V(v_i) - I$ at the 8 vertices [10327-lookup-table-and-ambiguity-handling].
    f.  Use the `edgeTable` and `triangleTable` from the parent article to determine the triangulation [10327-lookup-table-and-ambiguity-handling].
    g.  Generate new triangles.
    h.  Identify new boundary edges from the generated triangles.
    i.  If a new edge has not been visited, add it to the Growth Queue and mark it as visited.
3.  **Termination:** The algorithm terminates when the Growth Queue is empty, having traced the connected component of the isosurface.

**Parameters:**
*   $R_c$: The radius for particle lookup. The lookup grid cube size is chosen to be much coarser than the marching cubes grid to keep traversal volume small, but the cache ensures exact spherical culling [S1].

**Edge Cases and Pitfalls:**
*   **Disconnected Components:** The algorithm only extracts the connected component containing the seeds. Other components must be seeded separately.
*   **Ambiguity:** Standard MC ambiguities (cases 46, 47, 98, 99, 210, 213, 214, 215) still apply and must be handled using the parent article's ambiguity resolution strategies [10327-lookup-table-and-ambiguity-handling].
*   **Cache Consistency:** The 3D lookup cache must be updated efficiently as particles move. The method uses a point insert strategy for the cache [S1].

## Subtopics

### [Edge Growth Marching Cubes](9433-edge-growth-marching-cubes.md)
A variant that resolves topological ambiguities by growing the surface from seed triangles, ensuring consistency across adjacent cubes.

## Operating characteristics
*   **Time Complexity:** Depends on the number of triangles in the isosurface and the cost of particle lookup. The 3D lookup cache allows for exact lookups without the overhead of culling false positives from a cubic volume [S1].
*   **Memory Complexity:** Proportional to the number of particles and the size of the lookup cache. The cache uses a hash map, which can have higher overhead than a dense grid but avoids the memory waste of a coarse grid covering a large volume [S1].
*   **Performance:** In tests on a 3.2Ghz Intel Core 2 Duo, this approach was compared against Marching Slices with block subdivision and other methods [S1]. The exact lookup in the 3D cache avoids the 6.5x particle overestimation seen in approaches using a $3 \times 3 \times 3$ cubic lookup volume, reducing the number of floating-point distance calculations needed for culling [S1].
*   **Strengths:** Unbounded rendering; topological consistency; efficient particle lookup for sparse data.
*   **Limitations:** Only extracts connected components; requires efficient cache management for dynamic particles.

## Background
The method was introduced in the context of real-time particle isosurface extraction [S1]. It was developed to address the limitations of existing approaches that used either point insertion with a 3D lookup or 3D insertion with point lookup, which suffered from false positives in particle queries [S1]. The approach was compared against Marching Slices with block subdivision and a 2D projection 1D lookup particle cache [S1].

## Sources
[S1] Real-time particle isosurface extraction, https://doi.org/10.1145/1342250.1342256, fetched 2026-11-04
