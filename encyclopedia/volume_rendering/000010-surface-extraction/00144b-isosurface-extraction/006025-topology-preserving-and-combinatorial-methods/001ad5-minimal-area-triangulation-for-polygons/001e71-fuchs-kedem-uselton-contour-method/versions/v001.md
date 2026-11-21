# Fuchs-Kedem-Uselton Contour Method
Also called: Minimum-cost cycle contour method

## Overview
The Fuchs-Kedem-Uselton (FKU) Contour Method is an isosurface extraction algorithm that generates a minimum-area triangulated surface from volumetric data by solving a minimum-cost cycle problem in a directed graph. It distinguishes itself from heuristic approaches (like [Merging Triangulations](3936-merging-triangulations.md)) by guaranteeing an optimal solution that minimizes visual artifacts, at the cost of higher computational complexity.

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (volume data) and a target isovalue.
*   **Output:** A set of triangles representing the isosurface.

**Data Structures:**
*   **Contours:** A list of polygons, where each polygon represents the intersection of the isosurface with a single data slice. Each polygon consists of an ordered sequence of vertices.
*   **Directed-Toroidal Graph:** A graph constructed to represent possible triangulations between two adjacent contours.

**Steps:**
1.  **Preprocess (Intra-slice):** Generate contours within single slices of the data. This involves marching through the volume cells to identify edges where the scalar value crosses the isovalue, connecting these edge intersections to form closed polygons (contours) for each slice.
2.  **Form Inter-slice Triangles:** For every pair of adjacent slices containing contours, construct a directed-toroidal graph. The nodes of this graph represent the vertices of the two contours. Edges represent potential triangles connecting vertices from one contour to the other.
3.  **Solve Minimum-Cost Cycle:** Find the minimum-cost cycle in the directed-toroidal graph. The cost of a cycle corresponds to the total area of the triangles formed. The optimal solution to this problem yields the minimum-area surface triangulation between the two contours.
4.  **Render:** Pass the generated triangles to a conventional polygon rendering engine to create the image.

**Formulas:**
The bound for the maximum cost $\tau(m, n)$ of finding the optimal solution for two contours with $m$ and $n$ points respectively (where $m < n$) is:

$$ \tau(m, n) < (\lceil \log_2 m \rceil + 2) \times (2mn + m) $$

Where:
*   $m$ is the number of points in the smaller contour.
*   $n$ is the number of points in the larger contour.
*   $\lceil \cdot \rceil$ is the ceiling function.

**Parameters:**
*   **Isovalue:** The scalar threshold for surface extraction. Not given in the sources.
*   **Cost Function:** The area of the triangles. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources.
*   **Boundaries:** Not given in the sources.
*   **Degenerate Input:** Not given in the sources.

## Subtopics

## Operating characteristics
*   **Time Complexity:** The complexity is bounded by $\tau(m, n) < (\lceil \log_2 m \rceil + 2) \times (2mn + m)$ for $m < n$ [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Generates the minimum-area surface, reducing visual artifacts [S1, unofficial].
*   **Limitations:** Higher computational cost compared to heuristic methods [S1, unofficial].
*   **When to use:** When optimal surface quality is prioritized over speed [S1, unofficial].
*   **When not:** When computational speed is more important than optimal results [S1, unofficial].

## Background
The method was introduced by Fuchs, Kedem, and Uselton in 1977 [S1, unofficial]. It was the first widely used coercion method for isosurface extraction. It is used in volume rendering to generate isosurfaces from volumetric data.

## Sources
[S1] A Parallel, Feed-Forward Volume Rendering Algorithm, https://www.cs.unc.edu/techreports/91-029.pdf, fetched 2026-10-27 (unofficial)
