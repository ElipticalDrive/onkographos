# Multi-Resolution Points

Also called: Error-controlled Point Splatting, Reduced Point Cloud Rendering

## Overview
Multi-Resolution Points is a hierarchical splatting technique that renders a single large point in place of many small points to reduce rasterization cost and storage requirements [S1, unofficial]. It differs from standard splatting by dynamically controlling point size based on local volume content and error metrics, rather than using a fixed resolution. This builds on [Hierarchical and Multi-Resolution Splatting](5618-hierarchical-and-multi-resolution-splatting.md).

## How it works
**Inputs:** A set of 3D points with attributes (position, color, opacity, normal).
**Outputs:** A rendered image.

**Data Structures:**
1.  **Point Tree:** Points are organized into a hierarchical tree structure (e.g., octree or quadtree). Each node represents a cluster of points.
2.  **Visibility Map:** A quadtree used to track occlusion and cull nodes [S1, unofficial].

**Algorithm:**
1.  **Tree Construction:** Organize the input points into a tree structure. The leaves contain individual points or small clusters [S1, unofficial].
2.  **Error Metric Calculation:** For each node, calculate a local error metric to determine if the node can be represented by a single larger point. The sources cite the Root Mean Square (RMS) error metric from Laur and Hanrahan [S1, unofficial].
    *   Let $P_i$ be the original points in a node.
    *   Let $P_{avg}$ be the representative point (e.g., centroid) for the node.
    *   The error $E$ is calculated as the RMS deviation of the original points from the representative point, potentially weighted by opacity or other attributes.
    *   *Note:* Welsh [S1, unofficial] also mentions using a frequency-space metric to control error.
3.  **Threshold Comparison:** Compare the calculated error $E$ against a user-defined error threshold $T$.
    *   If $E \le T$, the node is collapsed into a single large point (the representative point).
    *   If $E > T$, the node is subdivided, and the process recurses to its children.
4.  **Occlusion Culling:** Use the visibility map to cull occluded nodes.
    *   Check if a node is occluded by previously rendered opaque geometry.
    *   If occluded, skip rasterization for this node and its children.
    *   If not occluded, proceed to rasterization.
5.  **Rasterization:** Rasterize the points that pass the error threshold and occlusion tests. Each point is splatted as a 2D primitive (e.g., a disk or Gaussian blob) onto the screen.
6.  **Visibility Update:** Update the visibility map with the newly rendered points to assist in culling subsequent nodes [S1, unofficial].

**Parameters:**
*   **Error Threshold ($T$):** User-set value to control the trade-off between accuracy and performance. Lower values result in more points (higher accuracy, slower); higher values result in fewer points (lower accuracy, faster) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Floating-point precision errors in error metric calculation may lead to incorrect culling or subdivision.
*   **Boundaries:** Points near the edge of the tree structure or scene bounds require careful handling to avoid artifacts.
*   **Degenerate Input:** Clusters of points with identical positions or attributes may cause division by zero or undefined behavior in error metrics if not handled.

## Subtopics

### [Cloud Impostor Rendering](5125-cloud-impostor-rendering.md)
A technique to accelerate cloud rendering by replacing geometric particle models with 2D texture impostors, exploiting the lack of high-frequency edges in clouds to tolerate lower resolution textures and frequent updates.

## Operating characteristics
*   **Time Complexity:** Depends on the tree depth and the error threshold. In the worst case, all points are rendered ($O(N)$). In the best case, large clusters are collapsed ($O(1)$ per cluster). The visibility map update adds overhead.
*   **Memory Complexity:** $O(N)$ for storing the point tree and visibility map. The tree structure adds overhead compared to a flat list.
*   **Performance:** Welsh [S1, unofficial] reported reducing 240k points to 90k points using a frequency-space metric, demonstrating significant reduction in rasterization cost.
*   **Strengths:** Reduces rasterization overhead by merging overlapping points; reduces storage by retaining only relevant points.
*   **Limitations:** Requires preprocessing to build the tree; error metric calculation can be expensive; occlusion culling adds complexity.
*   **When to Use:** Large point clouds where many points overlap or are occluded; scenes with varying density.
*   **When Not to Use:** Small point clouds where overhead dominates; scenes requiring high-frequency detail that cannot be captured by large splats.

## Background
The concept of organizing points into a tree and using error metrics to control resolution was introduced by Laur and Hanrahan [S1, unofficial]. Welsh [S1, unofficial] extended this with frequency-space metrics. The technique is part of the broader field of volumetric point rendering, aiming to balance visual fidelity with rendering performance. The sources also mention alternative grids (BCC, hexagonal) for more efficient point sampling, citing Conway [S1, unofficial], but these are related to compression rather than the core multi-resolution algorithm.

## Sources
[S1] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
