# Surface-following Marching Cubes with Blocking


## Overview
Surface-following Marching Cubes with Blocking is a surface extraction method designed for unbounded rendering of particle-based volumes [S1]. It resolves the topological inconsistencies of standard voxel-based Marching Cubes by treating the iso-surface as a connected manifold grown from seed triangles rather than processed independently per voxel [S1]. Unlike standard Marching Cubes, which evaluates each cube independently and may produce inconsistent topology at shared edges, this method propagates the surface through adjacent cubes via a queue of "growth edges," ensuring topological consistency across the entire reconstructed model [S1]. It is particularly suited for 3D reconstruction where only the main contour is required or for isolating specific connected components from scattered data by controlling the seed region [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of particles (points with associated scalar values or densities) and an isovalue threshold.
*   **Output:** A manifold triangle mesh representing the iso-surface.

**Data Structures:**
1.  **Particle List:** Stores particle positions and values.
2.  **Lookup Grid:** A separate grid for particle lookups with cube size $R_c$. This grid is much coarser than the marching cubes grid [S1].
3.  **Corner Cache:** A hash map storing the geometric corners (vertices) of the generated mesh to ensure shared vertices are reused [S1].
4.  **Growth Queue:** A queue of edges or cubes to process next, enabling the surface-following traversal [S1].

**Algorithm Steps:**
1.  **Initialization:** Identify seed triangles or cubes that intersect the isovalue. Add their boundary edges to the Growth Queue.
2.  **Surface Following:** While the Growth Queue is not empty:
    a.  Dequeue an edge or cube.
    b.  Determine the local iso-surface geometry using Marching Cubes logic on the relevant particles.
    c.  Generate triangles for the surface segment.
    d.  Identify new boundary edges created by these triangles.
    e.  Enqueue new edges if they have not been processed, ensuring each edge is processed once to maintain manifold consistency.
3.  **Particle Lookup Optimization:** To efficiently find particles near a cube, use the separate lookup grid with cube size $R_c$. This keeps the traversal volume to a $3 \times 3 \times 3$ block [S1]. This approach performs an exact lookup without returning particles outside $R_c$, avoiding the false positives inherent in coarser grids that return particles from a $(3R_c)^3$ volume which must then be culled by floating-point distance calculations [S1].
4.  **Mesh Construction:** Use the corner cache (hash map) to store and retrieve vertex positions. This ensures that shared vertices between adjacent triangles are identical, preserving the manifold property [S1].

**Parameters:**
*   $R_c$: The cube size of the particle lookup grid. It is chosen to be much coarser than the marching cubes grid to balance lookup speed and accuracy [S1].
*   Isovalue: The threshold for surface extraction.

**Edge Cases and Pitfalls:**
*   **Disconnected Components:** The surface-following nature means it only extracts the connected component containing the seeds. Disconnected parts of the volume will not be rendered unless additional seeds are provided [S1].
*   **Lookup Grid Resolution:** If the lookup grid resolution is increased to reduce false positives, the insertion/lookup costs quickly grow and dominate overall isosurface extraction costs [S1].

## Subtopics

### [Edge Growth Marching Cubes](9433-edge-growth-marching-cubes.md)
Edge Growth Marching Cubes is a surface extraction algorithm that resolves the topological ambiguities and disconnected holes inherent in standard Marching Cubes (MC) by treating the iso-surface as a connected manifold grown from seed triangles rather than processed voxel-by-voxel. Unlike standard MC, which evaluates each cube independently and may produce inconsistent topology at shared edges, this method propagates the surface through adjacent cubes via a queue of "growth edges," ensuring topological consistency across the entire reconstructed model. It is particularly suited for 3D reconstruction where only the main contour is required or for isolating specific connected components from scattered data by controlling the seed region.

## Operating characteristics
*   **Time Complexity:** The surface-following traversal ensures each edge is processed once, leading to efficient mesh generation. However, the particle lookup cost depends on the density of particles and the size of $R_c$. The $3 \times 3 \times 3$ traversal volume keeps lookup costs low compared to naive methods [S1].
*   **Memory Complexity:** Requires storage for the particle list, the coarse lookup grid, and the corner cache. The corner cache can grow large for complex surfaces but reduces duplicate vertex generation [S1].
*   **Performance:** In tests on a 3.2Ghz Intel Core 2 Duo processor with 4MB of L2 cache, this approach was compared against Marching Slices with block subdivision and other methods [S1]. The exact frame rates are not specified in the provided notes, but the method is designed for real-time particle isosurface extraction [S1].
*   **Strengths:** Topological consistency (manifold mesh), efficient particle lookup with coarse grid, suitable for unbounded rendering.
*   **Limitations:** Only extracts connected components from seeds; lookup grid resolution trade-off between accuracy and speed.

## Background
This method was introduced in the context of real-time particle isosurface extraction [S1]. It addresses the limitations of standard Marching Cubes when applied to particle-based data, where topological inconsistencies and disconnected holes are common. The use of a separate coarse grid for particle lookups is a key innovation to maintain performance while ensuring accurate surface reconstruction [S1].

## Sources
[S1] Real-time particle isosurface extraction, https://doi.org/10.1145/1342250.1342256, fetched 2026-11-04
