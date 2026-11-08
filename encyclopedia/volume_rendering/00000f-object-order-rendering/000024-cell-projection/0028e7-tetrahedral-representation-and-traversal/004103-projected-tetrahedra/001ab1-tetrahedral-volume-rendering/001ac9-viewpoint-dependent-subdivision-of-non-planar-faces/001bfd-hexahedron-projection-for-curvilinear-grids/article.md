# Hexahedron Projection for Curvilinear Grids

Also called: Hexahedron Projection

## Overview
Hexahedron Projection is a cell projection method for direct volume rendering of curvilinear hexahedral grids that classifies the 2D projection of each hexahedron into specific topological cases and renders them as pre-defined triangle fans or strips [S1, unofficial]. It solves the problem of efficiently rendering non-planar faces in hardware by avoiding the computational geometry complexity of general line-sweep subdivision or full tetrahedral decomposition [S1, unofficial]. Unlike its siblings, which may rely on view-independent classification or full subdivision into tetrahedra, this method uses viewpoint-dependent case analysis to minimize vertex and fragment operations, offering a significant speedup over subdividing all cells into five tetrahedra [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A curvilinear hexahedral grid. Each hexahedron is defined by 8 vertices $V_0 \dots V_7$. The vertices are numbered according to the standard hexahedron topology (Figure 2 in [S1, unofficial]).
*   **Output:** A stream of triangle fans and triangle strips suitable for hardware rasterization.

**Data Structures:**
*   **Hexahedron Vertices:** An array of 8 3D points.
*   **Projected Vertices:** The 2D screen-space coordinates of the 8 vertices.
*   **Case Classification:** An integer `count` (0, 1, 2, or 4) and a boolean flag for bow-tie/degenerate status.
*   **New Vertices:** Temporary 2D points ($V_8, V_9, \dots$) computed as intersections of projected edges.

**Algorithm Steps:**

1.  **Bow-Tie and Degeneracy Check:**
    For each hexahedron, iterate through its 6 quadrilateral faces. For each face, compute the line equations for its 4 projected edges. For each edge line, check if the other 2 vertices of that face lie on the same side of the line.
    *   If any vertex lies on a different side, the face projection is a self-intersecting "bow-tie".
    *   If any vertices coincide (degenerate cell).
    *   **Action:** Subdivide the hexahedron into 5 tetrahedra and render using Shirley-Tuchman triangle fans [S1, unofficial].

2.  **Containment Test (Count Calculation):**
    For non-degenerate, non-bow-tie cells, select a reference face (typically the one containing projected vertices). Compute line equations for the 4 edges of this reference face. Test the 4 non-face vertices against these 4 line equations to determine if they lie inside the projected quadrilateral.
    *   Let `count` be the number of contained vertices. Possible values: 0, 1, 2, 4. (A count of 3 implies a bow-tie, which is already handled) [S1, unofficial].

3.  **Vertex Permutation:**
    Permute the vertex indices of the hexahedron so that:
    *   The vertices of the reference face containing the projected vertices are mapped to indices $0, 1, 2, 3$.
    *   If `count` = 1, the contained vertex is mapped to index $7$.
    *   If `count` = 2, the contained vertices are mapped to indices $6$ and $7$ [S1, unofficial].

4.  **Case-Specific Rendering:**
    Based on the `count` and specific intersection availability, generate triangle fans/strips.

    **Case Count = 1 (Standard):**
    *   Find intersection $V_8$ of edges $V_7V_4$ and $V_0V_1$.
    *   Find intersection $V_9$ of edges $V_7V_6$ and $V_1V_2$.
    *   Fan 1: $8, 1, 5, 4, 0, 3, 7, 1$
    *   Fan 2: $9, 7, 3, 2, 6, 5, 1, 7$ [S1, unofficial]

    **Case Count = 1 (Alternate A):**
    *   Condition: $V_8$ not found at $V_7V_4/V_0V_1$.
    *   Find $V_8$ at intersection of $V_7V_4$ and $V_1V_2$.
    *   Find $V_{10}$ at intersection of $V_7V_4$ and $V_1V_5$.
    *   Fan 1: $9, 6, 2, 3, 7, 8, 10, 5, 6$
    *   Fan 2: $8, 7, 3, 0, 1$
    *   Fan 3: $10, 8, 1, 0, 4, 5$ [S1, unofficial]

    **Case Count = 1 (Alternate B):**
    *   Condition: $V_9$ not found at $V_7V_6/V_1V_5$.
    *   Find $V_9$ at intersection of $V_7V_6$ and $V_0V_1$.
    *   Find $V_{10}$ at intersection of $V_7V_6$ and $V_1V_5$.
    *   Fan 1: $10, 5, 6, 2, 1, 9, 8, 4, 5$
    *   Fan 2: $9, 1, 2, 3, 7$
    *   Fan 3: $8, 9, 7, 3, 0, 4$ [S1, unofficial]

    **Case Count = 1 (Alternate C):**
    *   Condition: $V_{10}$ not found at $V_7V_4/V_1V_5$ (after Alternate A logic).
    *   Find $V_{10}$ at intersection of $V_1V_2$ and $V_4V_0$.
    *   Fan 1: $9, 2, 3, 7, 8, 4, 5, 6, 2$
    *   Fan 2: $10, 7, 3, 0, 1, 5, 4, 8, 7$ [S1, unofficial]

    **Case Count = 1 (Alternate D):**
    *   Condition: $V_{10}$ not found at $V_7V_6/V_1V_5$ (after Alternate B logic).
    *   Find $V_{10}$ at intersection of $V_2V_6$ and $V_1V_5$.
    *   Fan 1: $8, 4, 0, 3, 7, 9, 6, 5, 4$
    *   Fan 2: $10, 6, 9, 7, 3, 2, 1, 5, 6$ [S1, unofficial]

    **Case Count = 4:**
    *   No extra vertices needed.
    *   Fan 1: $4, 0, 3, 7, 6, 5, 1, 0$
    *   Fan 2: $6, 7, 3, 2, 1, 5$ [S1, unofficial]

    **Case Count = 2:**
    *   Find $V_8$ at intersection of $V_7V_4$ and $V_0V_1$.
    *   Find $V_9$ at intersection of $V_6V_5$ and $V_0V_1$.
    *   Fan: $7, 8, 0, 3, 2, 6, 9, 8$
    *   Strip: The source text mentions a triangle strip is used but does not explicitly list the vertex indices for the strip in the Count=2 section, unlike the Count=0 section. It references [4] for the method. [S1, unofficial]

    **Case Count = 0:**
    *   The projected vertices form a convex octagon.
    *   Permute indices counter-clockwise around the octagon.
    *   If vertex 0 connects to vertex 5 (the $i-3$ case), renumber by replacing index $i$ with $8-i$.
    *   Find $V_8$ at intersection of $V_0V_3$ and $V_1V_6$.
    *   Find $V_9$ at intersection of $V_0V_3$ and $V_2V_5$.
    *   Find $V_{10}$ at intersection of $V_4V_7$ and $V_2V_5$.
    *   Find $V_{11}$ at intersection of $V_4V_7$ and $V_1V_6$.
    *   Fan: $8, 0, 1, 2, 9, 10, 11, 7, 0$
    *   Strip: $7, 6, 11, 5, 10, 4, 3, 9, 2$ [S1, unofficial]

**Edge Cases and Pitfalls:**
*   **Floating Point Precision:** General line-sweep methods suffer from inconsistent topological tests (e.g., point vs. line) due to finite precision. This method avoids this by using pre-classified triangle fans for specific projection topologies [S1, unofficial].
*   **Bow-Tie Faces:** Self-intersecting face projections make visibility sorting impossible for back-to-front compositing. These must be detected and handled via tetrahedral subdivision [S1, unofficial].
*   **Degenerate Cells:** Cells with coincident vertices (common in cylindrical/spherical coordinates) must be subdivided into tetrahedra [S1, unofficial].

## Subtopics

### [Projection-Based Ray Casting for Curvilinear Grids](1756-projection-based-ray-casting-for-curvilinear-grids.md)
A direct volume rendering technique that determines cell traversal by projecting cell faces onto the 2D image plane. It solves the problem of efficiently finding entry and exit points in irregular meshes by avoiding complex 3D geometric intersection tests, relying instead on 2D screen-space projection and ray-crossing tests. Unlike general traversal methods that require pre-computed topological structures, this method uses on-the-fly decomposition and 2D projection, making it suitable for irregular sampling densities.

### [Depthless Cell Face Projection](6692-depthless-cell-face-projection.md)
A projection-based direct volume rendering method for curvilinear grids where each data value is mapped to color and opacity, and cell faces are drawn as Gouraud-shaded polygons. Depth is not considered during compositing.

## Operating characteristics
*   **Time Complexity:** The method involves a constant number of geometric tests per hexahedron (bow-tie check, containment test, intersection calculations). The classification step dominates the CPU time.
*   **Memory Complexity:** Low. Requires storing the grid vertices and temporary intersection points. No large pre-computed topological structures are needed.
*   **Measured Performance:** On an 800 MHz Pentium4 Xeon and nVidia 5900FXUltra, rendering 141,960 hexahedra (including 50,352 degenerate and 924 bow-tie) took 5.27 seconds total: 0.13s data read, 3.5s case classification, 1.51s rendering [S1, unofficial].
*   **Scaling:** The method scales linearly with the number of hexahedra.
*   **Strengths:** Faster than subdividing all cells into five tetrahedra because it requires fewer vertex and fragment operations. Handles non-planar faces in curvilinear grids effectively.
*   **Limitations:** Requires handling many specific topological cases (standard and alternates for count=1). Bow-tie and degenerate cases still require tetrahedral subdivision.
*   **When to Use:** When rendering curvilinear hexahedral grids on hardware that supports triangle fans/strips, and performance is critical.
*   **When Not to Use:** When the grid has a high proportion of bow-tie or degenerate cells, or when a simpler, more robust (but slower) method like full tetrahedral subdivision is preferred for simplicity.

## Background
*   **Origin:** Proposed by Nelson Max at Lawrence Livermore National Laboratory [S1, unofficial].
*   **History:** The method builds on earlier work by Schussman and Max [4] for perspective views of cubes, extending it to curvilinear grids where faces are non-planar. It addresses the limitations of line-sweep methods by Wilhelms and Van Gelder [2] and incremental methods by Max, Williams, and Silva [3], which were difficult to implement robustly due to floating-point precision issues [S1, unofficial].
*   **Usage:** Used in direct volume rendering of curvilinear grids, particularly in scientific visualization contexts where performance on hardware is important.

## Sources
[S1] Hexahedron Projection for Curvilinear Grids (revision 1), https://www.osti.gov/biblio/862378, fetched 2026-10-26 (unofficial)
