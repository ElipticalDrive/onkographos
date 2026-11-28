# Marching Lines Algorithm

Also called: MLA

## Overview
The Marching Lines Algorithm (MLA) extracts 1D isocontour line segments from a discrete 2D scalar field by processing grid cells in a linear, row-by-row manner. It serves as the 1D analog to [Marching Squares](918-marching-squares.md) and [Marching Cubes](918-marching-cubes.md). Unlike template-based methods, MLA is not template-based and requires further processing to properly resolve apparent cell ambiguities [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 2D scalar field $f(x, y)$ defined on a regular grid of vertices.
*   **Output:** A set of line segments (edges) connecting points where the scalar field crosses a specified isovalue $I$.

**Data Structures:**
*   **Grid:** A 2D array of scalar values.
*   **Edge List:** A list of line segments, each defined by two endpoints $(x_1, y_1)$ and $(x_2, y_2)$.

**Steps:**
1.  **Iterate Rows:** Process the grid row by row (or column by column). For each row $y$, iterate through the cells between vertex $x$ and $x+1$.
2.  **Cell Evaluation:** For each cell, evaluate the scalar values at the four corners: $v_{00}, v_{10}, v_{01}, v_{11}$.
3.  **Sign Determination:** Determine the binary state of each corner relative to the isovalue $I$: $s_{ij} = \text{sign}(v_{ij} - I)$.
4.  **Ambiguity Handling:** Because MLA is not template-based, it does not use a precomputed lookup table for all 256 cases. Instead, it identifies "apparent cell ambiguities" [S1, unofficial]. These typically arise when the configuration of signs suggests multiple valid topological interpretations within a single cell or across adjacent cells.
5.  **Line Generation:** For non-ambiguous cases, generate line segments connecting the intersection points on the cell edges. The intersection point on an edge between vertices with values $v_a$ and $v_b$ is calculated via linear interpolation:
    $$ t = \frac{I - v_a}{v_b - v_a} $$
    $$ P = (1-t)P_a + tP_b $$
6.  **Post-Processing:** Apply further processing to resolve the apparent ambiguities identified in Step 4. This ensures topological consistency and prevents artifacts like holes or disconnected segments [S1, unofficial].

**Parameters:**
*   **Isovalue ($I$):** The scalar threshold for extraction.
*   **Grid Resolution:** Determines the granularity of the output.

**Edge Cases and Pitfalls:**
*   **Directional Ambiguities:** The algorithm may suffer from directional ambiguities if the processing order (row-major vs. column-major) affects the resolution of ambiguous cells [S1, unofficial].
*   **Precision:** Floating-point precision issues near the isovalue can lead to inconsistent sign determinations.
*   **Degenerate Input:** If a vertex value exactly equals the isovalue, special handling is required to avoid division by zero during interpolation.

## Subtopics

### Marching Squares on Triangular Meshes
Application of the Marching Squares algorithm to triangular meshes (connected triangles with data at vertices), such as those from Delaunay triangulation of scattered data points [S2]. A triangular cell is always planar (a 2-simplex), ensuring a unique linear interpolant across it and no possibility of an ambiguous saddle [S2]. For isolines, the analysis involves 3 binary digits at vertices, resulting in 8 possibilities [S2]. For isobands, it involves 3 ternary trits, resulting in 27 possibilities [S2].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of cells in the grid.
*   **Memory Complexity:** $O(N)$ for storing the grid and output segments.
*   **Strengths:** Simple implementation; low memory overhead.
*   **Limitations:** Requires post-processing to resolve ambiguities; not template-based, which can lead to inconsistencies if not handled carefully [S1, unofficial].
*   **When to Use:** Suitable for 2D scalar fields where simplicity and speed are prioritized over complex topological handling.

## Background
The Marching Lines Algorithm is a foundational technique in isosurface extraction, serving as the 1D counterpart to Marching Squares and Marching Cubes. It is part of the broader family of marching algorithms used in volume rendering and scientific visualization.

## Sources
[S1] Volume-Enclosing Surface Extraction, https://arxiv.org/abs/1011.1787v4, fetched 2026-10-26 (unofficial)
[S2] Marching squares, https://smunix.github.io/en.wikipedia.org/wiki/Marching_squares.html, fetched 2026-10-30
