# Wavelet Tree Overview Map

Also called: Overview Map Visualization

## Overview
The Wavelet Tree Overview Map is a hierarchical visualization technique designed to provide an instant overview of the error distribution across an entire wavelet tree and to highlight the current Level of Detail (LOD) cut. It solves the problem of navigating large multiresolution volume datasets by mapping the tree structure into a 2D geometric frame, allowing users to identify regions requiring refinement or coarsening. It differs from its sibling [Projecting Detail Coefficients](1802-projecting-detail-coefficients.md), which visualizes coefficient magnitudes directly, and [Wavelet Tree](5390-wavelet-tree.md), which represents the abstract data structure; the Overview Map specifically encodes spatial hierarchy and error magnitude into a continuous color field within a triangular frame [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The complete wavelet tree structure (nodes, levels, parent-child relationships) and the error value associated with each node.
*   **Output:** A 2D raster image representing the triangular frame, where color indicates error magnitude and geometry indicates tree position.

**Data Structures:**
The method relies on the standard wavelet tree hierarchy. Each node $N$ is defined by:
*   `level`: The depth in the tree (root is 0).
*   `index`: The ordinal position among siblings at the same level.
*   `error`: The scalar error metric associated with the node.
*   `children`: List of child nodes.

**Steps:**

1.  **Frame Construction:**
    Draw an equilateral triangle frame. The apex represents the root of the wavelet tree. The base represents the set of all leaf nodes. Draw horizontal line segments within the triangle at equal vertical distances to depict intermediate tree levels [S1].

2.  **Node Mapping:**
    Map each wavelet tree node to a specific point on the horizontal line segment corresponding to its tree level.
    *   Let $L$ be the total number of levels.
    *   For a node at level $l$ (where $0 \le l < L$), its vertical position $y$ is determined by the linear interpolation between the apex ($y=0$) and the base ($y=H$, where $H$ is the triangle height).
    *   The horizontal position $x$ is determined by the node's ordinal index among its siblings. All nodes at the same level are arranged equidistantly from left to right.
    *   This mapping is performed in data space, independent of the run-time volume orientation [S1].

3.  **Color Transformation (Optional but Recommended):**
    To better utilize the color range for hierarchies with large error variations, transform the raw error values $E$ before mapping to colors. The transformed error $E'$ is calculated as:
    $$ E' = \sqrt{E} \quad \text{or} \quad E' = \log(E) $$
    This transformation prevents high-error nodes from dominating the color scale while obscuring low-error variations [S1].

4.  **Triangle Fan Filling:**
    Fill the triangle frame by rendering "triangle fans" for all parent-child branches.
    *   For each parent node $P$ and its children $C_1, C_2, \dots$, draw a triangle fan where the vertices are the mapped points of $P$ and its children.
    *   Assign colors to the vertices based on their transformed error values (e.g., red for high error, violet for low error).
    *   Linearly interpolate the colors across the interior of each triangle fan [S1].

5.  **Quad Filling (Hole Correction):**
    Filling only the triangle fans leaves "holes" (quads) between neighboring parent nodes. Specifically, a quad is formed by two neighboring parent nodes ($V_1, V_4$) and their adjacent children ($V_2$ being the last child of $V_1$, and $V_3$ being the first child of $V_4$).
    *   Identify these empty quads.
    *   Fill each quad with colors linearly interpolated from the four vertex colors ($V_1, V_2, V_3, V_4$).
    *   This step eliminates the fractal appearance of the holes and provides a continuous overview of the error distribution [S1].

6.  **LOD Highlighting:**
    Highlight the current Level of Detail (LOD) as a horizontal cut through the triangle frame.
    *   Identify the nodes that constitute the current LOD.
    *   Draw each node along this cut as a small square at its mapped point location.
    *   Connect these squares following the Depth-First Search (DFS) order of the nodes in the current LOD [S1].

**Pitfalls:**
*   **Holes:** If the quad filling step is omitted, the "holes" between neighboring parent nodes create a fractal appearance that hinders the observation of error distribution [S1].
*   **Precision:** The mapping is static based on data space; changes in volume orientation do not affect the overview map's geometry [S1].

## Subtopics

### Treemap Visualization for Active Nodes
A space-filling alternative to the triangular overview map, designed to reduce clutter when selecting specific nodes.
*   **Definition:** Uses recursively subdivided display areas (alternating vertical/horizontal subdivisions) to present hierarchical information via visual attributes like color and size of bounding rectangles [S1].
*   **Step:** Display only the nodes in the current LOD (active nodes) rather than all wavelet tree nodes. This avoids sub-pixel resolution issues and clutter found in the standard overview map. The spatial arrangement of an active node is deduced from its tree level and ordinal among siblings. Error is color-coded, and level is size-coded in the corresponding rectangular region [S1].
*   **Pitfall:** Specifying a small error tolerance results in nodes with similar error values, leading to indistinguishable colors.
*   **Mitigation:** If colors are indistinguishable, use an alternate error metric for color mapping: the difference between the parent and child nodes' error values, excluding the maximum error value of the child nodes. This helps discern differences between active nodes belonging to the same parent [S1].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Provides an instant, global overview of error distribution across the entire wavelet tree. The continuous color field (after quad filling) makes it easy to spot high-error regions. It is independent of run-time volume orientation [S1].
*   **Limitations:** The standard overview map displays all nodes, which can lead to clutter and sub-pixel resolution issues when the tree is deep or the LOD is fine. The triangular geometry may make precise selection of individual nodes difficult compared to the Treemap variant [S1].
*   **When to use:** When a user needs a global view of the data's error distribution to guide initial LOD selection or navigation.
*   **When not to use:** When precise selection of specific subvolumes is required, or when the number of active nodes is small and high precision is needed (use Treemap instead) [S1].

## Background
The Wavelet Tree Overview Map was introduced as part of a Hierarchical Navigation Interface for volume rendering. It leverages the multiresolution properties of wavelet transforms to allow users to navigate the trade-off between rendering quality and performance. The method builds on the structure of the wavelet tree and uses visual encoding (geometry for hierarchy, color for error) to facilitate user interaction [S1].

## Sources
[S1] Hierarchical Navigation Interface: Leveraging Multiple Coordinated Views for Volume Rendering, https://academicweb.nd.edu/~cwang11/papers/iv05-hni.pdf, fetched 2026-10-30
