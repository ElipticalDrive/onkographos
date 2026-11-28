# Conservative Cage Mesh Decimation
Also called: Conservative Mesh Simplification, Inclusion-preserving Mesh Decimation

## Overview
Conservative Cage Mesh Decimation is a mesh simplification technique that reduces the vertex count of a surface mesh while strictly enforcing the inclusion of a target 3D selection volume. Unlike standard edge collapse methods that may shrink the mesh inside the selection boundary, this method adds linear constraints to guarantee that the simplified "cage" always encloses the original region. It is distinguished from [Hoppe's Progressive Mesh Simplification](14926-hoppe's-progressive-mesh-simplification.md) by its hard geometric constraints rather than purely error-based metrics, and from [Mesh Quality Maintenance via Edge Collapse/Split](21387-mesh-quality-maintenance-via-edge-collapse-split.md) by its specific focus on volume preservation for interactive editing.

## How it works
The method simplifies a surface mesh extracted from a binary grid representation of a user selection. It operates by iteratively collapsing edges that satisfy a set of geometric and topological constraints.

**Inputs and Outputs:**
*   **Input:** A 3D binary grid indicating the simplified selection volume.
*   **Output:** A simplified triangular mesh (the cage) that strictly encloses the selection volume.

**Data Structures:**
*   **Binary Grid:** A 3D array of voxels where `1` indicates inclusion in the selection and `0` indicates exclusion.
*   **Triangle Compactness Metric:** A scalar value measuring the shape quality of a triangle, used to reject collapses that create thin triangles. (Exact formula not provided in sources; referenced as "Appendix A" in [S1]).

**Algorithm Steps:**
1.  **Initial Mesh Extraction:** Generate an initial surface mesh from the input binary grid using the Marching Cubes algorithm [Lorensen and Cline 1987].
2.  **Candidate Evaluation:** For each candidate edge collapse, evaluate the following conditions:
    *   **Inclusion Constraint:** Apply a linear constraint to ensure the collapse does not violate the strict inclusion of the selected volume. This is implemented using the "Progressive Hulls" algorithm [Sander et al. 2000], which solves a quadratic programming problem based on the dual method [Goldfarb and Idnani 1983] to maintain the hull property.
    *   **Manifold Check:** Test the canonical link condition [Dey et al. 1998] to ensure the mesh remains a manifold after the collapse.
    *   **Valence Bound:** Check the valence (number of connected edges) of the resulting vertex. Reject the collapse if the valence exceeds the user-defined upper bound.
    *   **Compactness Check:** Measure the compactness of the triangles involved in the collapse. Reject the collapse if it increases the compactness of any involved triangle, thereby preventing the formation of thin, degenerate triangles.
3.  **Edge Collapse:** If all conditions are met, perform the edge collapse.
4.  **Post-Processing:** Apply MeshFix [Attene et al. 2013] to remove holes, self-intersections, and degenerate elements. Note that this step may slightly alter the cage volume [G].

**Parameters:**
*   **Valence Bound:** The maximum allowed valence for any vertex. Default value is 12, but it is user-adjustable [S1].

**Pitfalls:**
*   **Volume Alteration:** The MeshFix post-processing step may slightly alter the cage volume, potentially causing the final mesh to no longer perfectly bound the selection [S1]. In rare cases where this causes issues, manual adjustment is required.

## Subtopics

### Cage Building Geometry Processing
This variant refers to the specific implementation details used in NeRFshop for constructing the initial cage and performing the simplification. It utilizes the C++ library libigl [Jacobson et al. 2018] with direct bindings to TetGen [Si 2015] for tetrahedralization. The simplification itself relies on an existing implementation of the "Progressive Hulls" algorithm [Sander et al. 2000] and a reimplementation of the quadratic programming solver by Goldfarb and Idnani [Goldfarb and Idnani 1983] based on the dual method. This setup is designed to support interactive editing within a user interface built with Dear ImGui and OpenGL.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** The method is designed to enable intuitive editing and interactivity, though specific performance metrics (FPS, simplification time) are not provided in the sources.
*   **Strengths:** Guarantees strict inclusion of the selection volume; prevents unintuitive editing behavior by limiting valence; prevents thin triangles via compactness checks.
*   **Limitations:** Post-processing with MeshFix can violate the strict inclusion property; requires quadratic programming solver for inclusion constraints.
*   **When to use:** When a simplified mesh is needed for interactive editing or tetrahedralization where the mesh must strictly enclose a specific volume.
*   **When not to use:** When perfect volume preservation is required after post-processing, or when high-valence vertices are acceptable.

## Background
The method was introduced in the context of NeRFshop, a system for interactive neural radiance field editing. It addresses the problem that naive mesh simplification (e.g., Garland and Heckbert 1997) violates the property that the cage encloses the selected region. The approach builds on standard practices for constrained mesh simplification, specifically citing Deng et al. 2011, Platis and Theoharis 2003, and Sander et al. 2000. The inclusion of the canonical link condition is inspired by Platis and Theoharis 2003. The implementation uses libigl and TetGen for geometry processing.

## Sources
[S1] NeRFshop, https://doi.org/10.1145/3585499, fetched 2026-10-26
