# Surface-following Marching Cubes with Blocking


## Overview
Surface-following Marching Cubes with Blocking is a surface extraction method designed for unbounded rendering of particle-based volumes [S1]. It differs from standard voxel-based Marching Cubes by employing a blocking algorithm and a 3D lookup cache for particle insertion, rather than evaluating each cube independently [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of particles (positions and scalar values) and an isovalue threshold.
*   **Output:** A triangle mesh representing the iso-surface.

**Data Structures:**
1.  **Marching Grid:** The primary structured 3D grid for surface extraction.
2.  **Lookup Grid:** A separate grid for particle lookups with cube size $R_c$, which is much coarser than the marching cubes grid [S1].
3.  **3D Lookup Cache:** Stores particle references to enable fast point insertion and retrieval [S1].

**Algorithm Steps:**
1.  Initialize the marching grid and the coarse lookup grid with cube size $R_c$ [S1].
2.  For each marching cube, query the lookup grid to find nearby particles within the blocking radius [S1].
3.  Compute the case index and generate triangles using standard Marching Cubes logic [G].
4.  Update the 3D lookup cache and particle insertions to maintain the unbounded volume [S1].
*Comment: The sources do not provide the full traversal or blocking algorithm details.*

**Parameters:**
*   $R_c$: The cube size of the particle lookup grid, chosen to be much coarser than the marching cubes grid to keep the traversal volume to $3 \times 3 \times 3$ [S1].
*   Isovalue: The threshold for surface extraction.

**Edge Cases and Pitfalls:**
*   **Lookup Grid Resolution:** Increasing the lookup grid resolution to reduce false positives causes insertion and lookup costs to quickly grow and dominate overall isosurface extraction costs [S1].
*   **Traversal Volume:** Using a coarse grid for lookups keeps the traversal volume to $3 \times 3 \times 3$, but alternative approaches using this grid return particles from a $(3R_c)^3$ volume that must be culled by floating-point distance calculations [S1].

## Subtopics

### [Edge Growth Marching Cubes](9433-edge-growth-marching-cubes.md)
Resolves topological ambiguities by growing the iso-surface from seed triangles via a growth queue, ensuring manifold consistency across the model. Chosen when topological correctness for connected components is critical.

## Operating characteristics
*   **Time Complexity:** Lookup cost depends on particle density and $R_c$ [S1]. The $3 \times 3 \times 3$ traversal volume keeps lookup costs low compared to naive methods [S1].
*   **Memory Complexity:** Requires storage for the particle list, the coarse lookup grid, and the 3D lookup cache [S1].
*   **Performance:** Tested on a 3.2 GHz Intel Core 2 Duo processor with 4 MB of L2 cache using a single core [S1]. The method is designed for real-time particle isosurface extraction [S1].
*   **Strengths:** Enables unbounded rendering, efficient particle lookup via a coarse grid, and avoids returning particles outside $R_c$ [S1].
*   **Limitations:** Lookup grid resolution involves a trade-off between accuracy and speed; higher resolution causes lookup costs to dominate [S1].

## Background
This method was introduced in the context of real-time particle isosurface extraction [S1]. It addresses limitations of standard Marching Cubes on particle data by using a separate coarse grid for lookups to maintain performance while enabling unbounded rendering [S1].

## Sources
[S1] Real-time particle isosurface extraction, https://doi.org/10.1145/1342250.1342256, fetched 2026-11-04
