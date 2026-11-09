# Wavelet Tree Overview Map

Also called: Overview Map Visualization

## Overview
The Wavelet Tree Overview Map is a hierarchical visualization technique designed to provide an instant, global overview of error distribution across a wavelet-based multiresolution volume dataset [S1]. It solves the problem of navigating complex wavelet trees by mapping tree nodes to a 2D triangular frame, allowing users to identify regions requiring higher or lower Levels of Detail (LOD) for rendering [S1]. It differs from [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md), which visualizes coefficient magnitudes directly, and [Wavelet Tree](5390-wavelet-tree.md), which typically represents structure via standard tree diagrams; the Overview Map specifically encodes spatial hierarchy and error magnitude into a continuous color field within a fixed geometric frame [S1].

## How it works
**Inputs:** The complete wavelet tree structure (nodes, parent-child relationships, tree levels) and the associated error metric for each node [S1].
**Outputs:** A 2D image representing the triangular frame with color-filled regions and an LOD cut indicator [S1].

**Data Structures:**
*   **Tree Node:** Contains level index, ordinal position among siblings, and error value [S1].
*   **Frame Geometry:** An equilateral triangle where the apex is the root (level 0) and the base contains all leaves [S1].

**Steps:**
1.  **Frame Construction:** Define an equilateral triangle frame. The apex represents the root of the wavelet tree. The base represents the set of all leaf nodes [S1].
2.  **Level Segmentation:** Draw horizontal line segments within the triangle frame at equal vertical distances to depict intermediate tree levels [S1].
3.  **Node Mapping:** Map each wavelet tree node to a specific point on the horizontal line segment corresponding to its tree level.
    *   Nodes at the same level are arranged equidistantly from left to right on their respective line segment [S1].
    *   This mapping is performed in data space, independent of the run-time volume orientation [S1].
4.  **Error Transformation (Optional):** To better utilize colors for hierarchies with large error ranges, transform the raw error values $E$ before mapping. Common transformations include:
    $$ E' = \sqrt{E} \quad \text{or} \quad E' = \log(E) $$
    [S1].
5.  **Color Mapping:** Map the (transformed) error values to colors. Typically, red indicates high errors and violet indicates low errors [S1].
6.  **Triangle Fan Filling:** Fill the triangle frame by rendering small triangle fans for all parent-child branches.
    *   Each fan vertex is a parent node, and the remaining vertices are its children [S1].
    *   Colors within each fan are linearly interpolated between the vertices [S1].
7.  **Hole Filling:** The initial fan filling leaves "holes" (quads) between neighboring parent nodes. A quad is formed by two neighboring parent nodes ($V_1, V_4$) and their adjacent children ($V_2, V_3$) [S1].
    *   Fill these quads with colors linearly interpolated from their four vertices to prevent a fractal appearance that hinders error observation [S1].
8.  **LOD Highlighting:** Highlight the current Level of Detail (LOD) as a horizontal cut through the triangle frame [S1].
    *   Draw each node along this cut as a small square indicating its point location [S1].
    *   Connect these squares following the Depth-First Search (DFS) order of the nodes [S1].

**Pitfalls:**
*   **Holes:** Filling only triangle fans leaves empty quads between neighboring parents, creating a fractal appearance that obscures error distribution [S1]. These must be explicitly filled [S1].
*   **Precision:** The mapping is static in data space; changes in volume orientation do not affect the map's geometry [S1].

## Subtopics

### Treemap Visualization for Active Nodes
A space-filling alternative that uses recursively subdivided display areas (alternating vertical/horizontal subdivisions) to present hierarchical information via visual attributes like color and size of bounding rectangles [S1]. Unlike the Overview Map, which shows all nodes, the Treemap displays only the nodes in the current LOD (active nodes) to avoid clutter and sub-pixel resolution issues [S1]. The spatial arrangement of an active node is deduced from its tree level and ordinal among siblings. Error is color-coded, and level is size-coded in the rectangular region [S1]. If colors are indistinguishable due to small error tolerances (similar error values), an alternate error metric is used: the difference between the parent and child nodes' error values, excluding the maximum error value of the child nodes [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides an instant overview of the entire wavelet tree's error distribution; independent of run-time volume orientation; serves as a starting point for local LOD adjustments [S1].
*   **Limitations:** The fractal appearance of unfilled holes hinders observation; showing all nodes can lead to clutter and sub-pixel resolution issues, necessitating the Treemap variant for active node selection [S1].
*   **When to use:** When a global view of error distribution is needed to guide LOD selection [S1].
*   **When not to use:** When precise selection of specific active nodes is required, as the Overview Map may suffer from sub-pixel resolution issues; use the Treemap variant instead [S1].

## Background
The Wavelet Tree Overview Map and its Treemap variant were introduced in the context of Hierarchical Navigation Interfaces for volume rendering [S1]. The method leverages the structure of wavelet-based multiresolution data to create a coordinated visualization that aids users in navigating the hierarchy and selecting appropriate LODs for rendering [S1].

## Sources
[S1] Hierarchical Navigation Interface: Leveraging Multiple Coordinated Views, https://academicweb.nd.edu/~cwang11/papers/iv05-hni.pdf, fetched 2026-10-30
