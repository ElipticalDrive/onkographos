# Wavelet Tree Overview Map

Also called: Overview Map Visualization

## Overview
The Wavelet Tree Overview Map is a hierarchical visualization technique designed to provide an instant, global overview of the error distribution within a wavelet-based multiresolution volume representation [S1]. It solves the problem of navigating complex LOD hierarchies by mapping the entire wavelet tree structure into a 2D equilateral triangle frame, allowing users to identify regions requiring refinement or coarsening before rendering [S1]. Unlike [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md), which projects coefficients directly, or [Wavelet Tree](5390-wavelet-tree.md), which focuses on the data structure itself, the Overview Map emphasizes spatial error distribution and LOD selection guidance [S1].

## How it works
The method maps the hierarchical wavelet tree nodes into a 2D geometric frame to visualize error metrics and current LOD states.

**Inputs and Outputs:**
*   **Input:** The complete wavelet tree structure (nodes, levels, parent-child relationships) and the hierarchical error metric for each node [S1].
*   **Output:** A 2D image where the triangle frame represents the tree, filled with interpolated colors representing error, and a highlighted "cut" representing the current LOD [S1].

**Data Structures:**
The wavelet tree is stored as a hierarchical structure where each node contains:
*   `level`: The depth in the tree (root is 0).
*   `ordinal`: The position index among siblings at the same level.
*   `error`: The hierarchical error value associated with the node [S1].
*   `children`: List of child nodes (for internal nodes).

**Steps:**

1.  **Frame Construction:** Draw an equilateral triangle frame. The apex represents the root of the wavelet tree. The base represents all leaf nodes [S1].
2.  **Level Segmentation:** Draw equal-distance horizontal line segments within the triangle frame. Each segment corresponds to a specific tree level [S1].
3.  **Node Mapping:** Map each wavelet tree node to a point on the horizontal line segment corresponding to its level.
    *   Nodes at the same level are arranged equidistantly from left to right based on their ordinal position in data space [S1].
    *   This mapping is independent of the run-time volume orientation [S1].
4.  **Error Color Transformation:** To handle large error ranges effectively, transform the raw error values $E$ before color mapping:
    $$ E' = \sqrt{E} \quad \text{or} \quad E' = \log(E) $$
    [S1].
5.  **Triangle Fan Filling:** Fill the triangle frame by rendering "triangle fans" for every parent-child branch.
    *   A triangle fan consists of a parent node vertex and its children vertices.
    *   Colors are linearly interpolated across the fan based on the transformed error values of the vertices [S1].
    *   High errors are mapped to red; low errors to violet [S1].
6.  **Hole Filling:** Identify "holes" (quads) formed between neighboring triangle fans. A quad is defined by two neighboring parent nodes ($V_1, V_4$) and their adjacent children ($V_2$ from $V_1$, $V_3$ from $V_4$) [S1].
    *   Fill these quads with colors linearly interpolated from the four vertices ($V_1, V_2, V_3, V_4$) [S1].
    *   This prevents a fractal appearance and ensures continuous error observation [S1].
7.  **LOD Highlighting:** Highlight the current Level of Detail (LOD) as a horizontal cut through the triangle frame.
    *   Draw each node along the cut as a small square at its mapped point location [S1].
    *   Connect these squares following the Depth-First Search (DFS) order of the nodes [S1].

**Pitfalls:**
*   **Holes:** Filling only triangle fans leaves quads empty, creating a fractal appearance that hinders error observation [S1].
*   **Color Indistinguishability:** If the error range is small, linear mapping may result in indistinguishable colors. Using square root or logarithmic transformation mitigates this [S1].

## Subtopics

### Treemap Visualization for Active Nodes
A space-filling alternative that displays only the active nodes (those in the current LOD) to avoid clutter and sub-pixel resolution issues inherent in the Overview Map [S1].
*   **Definition:** Uses recursively subdivided display areas (alternating vertical/horizontal subdivisions) to present hierarchical information via visual attributes like color and size of bounding rectangles [S1].
*   **Step:**
    1.  Display only nodes in the current LOD [S1].
    2.  Deduce spatial arrangement from tree level and ordinal among siblings [S1].
    3.  Color-code the error and size-code the level in the corresponding rectangular region [S1].
    4.  If colors are indistinguishable due to small error tolerance, use an alternate error metric: the difference between parent and child error values, excluding the maximum error value of the child nodes [S1].
*   **Pitfall:** Specifying a small error tolerance results in nodes with similar error values, leading to indistinguishable colors [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides an instant overview of error distribution; guides LOD selection; independent of run-time volume orientation [S1].
*   **Limitations:** The Overview Map displays all nodes, which can clutter the view and cause sub-pixel resolution issues for deep trees [S1]. The Treemap variant suffers from indistinguishable colors if error tolerances are too small [S1].
*   **When to use:** When a user needs a global view of the entire wavelet tree's error distribution to make initial LOD decisions [S1].
*   **When not to use:** When focusing on specific regions of interest or when the tree is too deep for the Overview Map to remain legible; use the Treemap variant or brushing/linking interfaces instead [S1].

## Background
The Wavelet Tree Overview Map and its variants were introduced in the context of hierarchical navigation interfaces for volume rendering. The method leverages the hierarchical nature of wavelet-based multiresolution data to create a visual guide for LOD selection. It is part of a broader effort to improve user interaction with complex multiresolution data structures, often combined with brushing and linking techniques for local adjustments [S1].

## Sources
[S1] Hierarchical Navigation Interface: Leveraging Multiple Coordinated Views, https://academicweb.nd.edu/~cwang11/papers/iv05-hni.pdf, fetched 2026-10-30
