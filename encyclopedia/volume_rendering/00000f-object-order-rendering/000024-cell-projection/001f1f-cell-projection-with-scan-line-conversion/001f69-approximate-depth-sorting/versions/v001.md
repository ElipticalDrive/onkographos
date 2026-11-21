# Approximate Depth Sorting

Also called: Depth Sorting, Depth Sorting for Blocks

## Overview
Approximate Depth Sorting is a heuristic ordering method used in cell projection volume rendering to process scan conversion in a depth-sorted sequence. It solves the problem of determining the correct front-to-back order of cells to reduce ray-segment list length and manipulation cost, which facilitates optimizations like Early Ray Termination [S2, unofficial]. It differs from exact methods (such as octree searches) by being simpler and less costly, accepting that perfect depth order is impossible due to pixel-by-pixel variations, while introducing no quality error in the final image [S2, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** A set of volume cells (blocks) defined by their geometric centers.
*   **Output:** A sorted list of cells ordered by their approximate distance from the camera.

**Data Structures**
*   **Cell Center:** A 3D coordinate $(x, y, z)$ representing the center of gravity of a cell.
*   **Distance Metric:** A scalar value derived from the 2D projection of the center.

**Steps**
1.  For each cell $i$, retrieve its center of gravity coordinates $(x_i, y_i, z_i)$.
2.  Compute the approximate depth $d_i$ using only the $x$ and $y$ components of the center relative to the camera position. As the scene segmentation occurs solely in the $xy$ plane without division along the $z$-axis, the $z$-component is ignored for sorting purposes [S1].
    $$ d_i = \sqrt{(x_i - x_{cam})^2 + (y_i - y_{cam})^2} $$
    Where $(x_{cam}, y_{cam})$ is the camera's position in the $xy$ plane.
3.  Sort the cells in ascending order of $d_i$ (closest to furthest).
4.  Use this sorted order for scan conversion and ray-segment composition [S2, unofficial].

**Parameters**
*   **Sorting Criterion:** Center of gravity depth.
*   **Components Used:** $x$ and $y$ only.

**Edge Cases and Pitfalls**
*   **Viewpoint Dependency:** The depth order of cells may change pixel-by-pixel in general. Therefore, the sorting must be recalculated every time the viewpoint changes [S2, unofficial].
*   **Approximation Error:** Perfect depth order is impossible to determine for all pixels simultaneously. This method accepts this limitation to avoid the computational cost of sorting and list manipulation required for near-perfect order (as in octree searches) [S2, unofficial].
*   **Quality Impact:** Despite being an approximation, it does not cause any quality error in the final image [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \log N)$ for sorting $N$ cells, plus $O(N)$ for distance calculation. Significantly less costly than exact depth sorting or octree traversal [S2, unofficial].
*   **Memory Complexity:** $O(N)$ to store the sorted list.
*   **Strengths:** Simple implementation, low computational overhead, enables efficient early ray termination by reducing ray-segment list length and manipulation cost [S2, unofficial].
*   **Limitations:** Order is only approximate; may not be optimal for all pixels, especially with complex occlusion. Requires recalculation on viewpoint change.
*   **When to use:** When rendering speed is critical and the scene segmentation is planar (xy-plane), or when the cost of exact sorting outweighs the marginal benefit of perfect order [S1][S2, unofficial].

## Background
The method was described in the context of parallel volume rendering with early ray termination. It was proposed as a solution to the difficulty of determining perfect depth order for cells, which may change pixel-by-pixel [S2, unofficial]. The approach leverages the fact that scene segmentation often occurs solely in the $xy$ plane, allowing $z$-components to be ignored for sorting [S1].

## Sources
[S1] City-on-Web: Real-Time Neural Rendering of Large-Scale Scenes on the Web, https://doi.org/10.1007/978-3-031-72970-6_22, fetched 2026-10-26
[S2] PDF Cell-projection Parallel Volume Rendering With Early Ray Termination, http://www.tomitaken.org/project/papers/ReVolver/2004/VIIP.pdf, fetched 2026-10-27 (unofficial)
