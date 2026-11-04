# MPVONC

Also called: MPVO Non-Convex

## Overview
MPVONC is a heuristic visibility sorting algorithm for cell projection volume rendering that extends the MPVO method to non-convex meshes [S1, S2, unofficial]. It solves the problem of ordering non-convex cells back-to-front by augmenting the partial ordering of MPVO with a centroid-based sort of exterior front-facing cells, followed by a depth-first search (DFS) [S1, S2, unofficial]. Unlike exact methods such as SXMPVO or A-buffer sorting, MPVONC is significantly faster but is approximate; it may produce visual artifacts due to sorting errors when occlusion relationships are not captured by the topological graph [S1, S2, unofficial].

## How it works
**Inputs:**
*   A set of polyhedral cells forming a non-convex mesh.
*   A viewpoint position $V$.
*   The directed graph $G$ of "behind" relationships derived from MPVO (edges point from occluded to occluding cells across shared faces) [S1, S2, unofficial].

**Outputs:**
*   A total ordering of cells suitable for back-to-front volume rendering.

**Data Structures:**
*   **Graph $G=(C, E)$**: Nodes $C$ are cells. Edges $E$ represent the MPVO partial ordering.
*   **Exterior Front-Facing Set $S_{ext}$**: A subset of cells $c \in C$ that have at least one face visible from the viewpoint (exterior front-facing).
*   **Centroid Distance**: For each cell $c$, the Euclidean distance $d(c, V)$ between its centroid $C_c$ and the viewpoint $V$.

**Steps:**
1.  **Identify Exterior Front-Facing Cells**: Determine the set $S_{ext}$ of all cells that have at least one face that is front-facing with respect to the viewpoint [S1, S2, unofficial].
2.  **Sort Exterior Cells**: Sort the cells in $S_{ext}$ in decreasing order of their centroid distance to the viewpoint:
    $$ d(c_i, V) \ge d(c_j, V) \implies c_i \text{ precedes } c_j $$
    Let the sorted list be $L = [c_1, c_2, \dots, c_k]$ [S1, S2, unofficial].
3.  **Depth-First Search (DFS)**: Initialize an empty list `OrderedCells` [G]. Iterate through each cell $c$ in $L$:
    *   If $c$ has not been visited in the current traversal or globally:
        *   Perform a DFS starting from $c$ in the graph $G$.
        *   Append visited nodes to `OrderedCells` in post-order (or as they are finalized by the DFS) to ensure back-to-front order relative to the graph edges [S1, S2, unofficial].
        *   Mark visited nodes to avoid re-processing [G].
4.  **Result**: The `OrderedCells` list constitutes the total visibility ordering [S1, S2, unofficial].

**Pitfalls:**
*   **Sorting Errors**: MPVONC may fail to produce a correct ordering if one cell occludes another across a region where ray-gaps are possible [S1, S2, unofficial]. This occurs specifically when the occluding cell has a centroid farther from the viewpoint than the occluded cell's centroid [S1, S2, unofficial]. In such cases, the heuristic sort places the occluded cell before the occluding one, leading to visual artifacts [S1, S2, unofficial].
*   **Non-Convexity**: The method assumes the underlying MPVO graph is valid. If the mesh is highly non-convex or contains complex topological features not captured by face-sharing relationships, the heuristic may not resolve all occlusions [S1, S2, unofficial].

## Subtopics

### Silva et al. Sweep Algorithm
A variant that removes the assumption that the mesh is convex by combining a sweep algorithm with the MPVO method [S1, unofficial]. It is substantially faster than previous exact methods for general meshes [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Very fast, comparable to MPVO [S1, S2, unofficial]. The sorting step is $O(N \log N)$ for exterior cells, and DFS is $O(N+E)$ [G].
*   **Accuracy**: Approximate. It usually generates a correct visibility ordering but is not guaranteed to be exact [S1, S2, unofficial].
*   **Strengths**: Simple to implement, very fast, handles non-convex meshes better than standard MPVO [S1, S2, unofficial].
*   **Limitations**: Prone to sorting errors and visual artifacts in specific non-convex configurations where centroid distance does not correlate with occlusion [S1, S2, unofficial]. Not suitable when artifact-free rendering is required.

## Background
MPVONC was described by Williams in [27] as a heuristic to extend the MPVO algorithm to non-convex meshes [S1, S2, unofficial]. It was developed to address the limitations of MPVO, which only computes an approximate ordering for general (non-convex) datasets, resulting in visual artifacts [S1, unofficial]. The method augments the MPVO partial ordering with a centroid-based sort and DFS [S1, S2, unofficial]. It is cited in works discussing image-space visibility ordering for cell projection volume rendering of unstructured data [S1, S2, unofficial].

## Sources
[S1] Image-space visibility ordering for cell projection volume rendering of unstructured data, https://www.researchgate.net/publication/8193687_Image-space_visibility_ordering_for_cell_projection_volume_rendering_of_unstructured_data, fetched 2026-10-29 (unofficial)
[S2] PDF 0078-0903-2 1..13, https://sci.utah.edu/~csilva/papers/tvcg2004cr.pdf, fetched 2026-10-28 (unofficial)
