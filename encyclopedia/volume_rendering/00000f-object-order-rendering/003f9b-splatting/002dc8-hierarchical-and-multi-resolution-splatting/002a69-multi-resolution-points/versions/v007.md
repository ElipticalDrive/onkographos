# Multi-Resolution Points

Also called: Error-controlled Point Splatting, Reduced Point Cloud Rendering

## Overview
Multi-Resolution Points is a technique that renders a single large point in place of many small points to reduce rasterization cost and storage requirements [S1, unofficial]. It solves the problem of rendering dense point clouds efficiently by adaptively selecting point sizes based on local error metrics [S1, unofficial]. Unlike standard splatting which treats all points equally, this method builds on [Hierarchical and Multi-Resolution Splatting](5618-hierarchical-and-multi-resolution-splatting.md) by using a tree structure to merge nearby points into a single representative primitive when the local error is below a user-defined threshold [S1, unofficial].

## How it works
**Inputs:** A set of points $P = \{p_i\}$, where each point $p_i$ has position $\mathbf{x}_i$, radius $r_i$, and scalar value $v_i$ (e.g., density or color).
**Outputs:** A reduced set of splatted points for rasterization.
**Data Structures:** An octree (or similar spatial tree) where each node stores the aggregate properties (mean position, total weight, mean scalar value) of its children [S1, unofficial].

1.  **Tree Construction:** Organize the input points into a hierarchical tree structure (e.g., octree) [S1, unofficial].
2.  **Error Computation:** For each internal node, compute a local error metric comparing the node's aggregate properties to those of its children. The standard metric is the Root Mean Square (RMS) error [S1, unofficial]:
    $$ E = \sqrt{\frac{\sum w_i (v_i - A)^2}{\sum w_i}} $$
    where $A$ is the aggregate scalar value of the node, $v_i$ are the scalar values of the children, and $w_i$ are their weights [S1, unofficial].
3.  **Adaptive Traversal:** Traverse the tree from root to leaves. A node is selected as a leaf (and thus rendered as a single large point) if its error $E$ is below a user-set threshold $\epsilon$ [S1, unofficial]. Otherwise, recurse to children [S1, unofficial].
4.  **Primitive Aggregation:** When a node is selected, its children are merged into a single representative point. The position is the weighted average of children positions, and the radius is determined by the spatial extent of the children [S1, unofficial].
5.  **Rasterization:** Rasterize the selected points using standard splatting techniques [S1, unofficial].

**Parameters:**
*   $\epsilon$: Error threshold. Controls the trade-off between accuracy and performance. [S1, unofficial]

**Edge Cases:**
*   **Precision:** Floating-point errors in RMS calculation may cause instability near the threshold. [G]
*   **Boundaries:** Points near the edge of the tree may have fewer children, affecting error estimation. [G]

## Subtopics

### [Cloud Impostor Rendering](5125-cloud-impostor-rendering.md)
Cloud Impostor Rendering is a technique to accelerate cloud rendering by replacing geometric particle models with 2D texture impostors, exploiting the lack of high-frequency edges in clouds to tolerate lower resolution textures and frequent updates. Unlike geometric models which require high-resolution impostors only at a distance, this method allows impostors to be used for clouds even when the viewer is inside or very close to them, significantly reducing overdraw and fill-rate requirements. It builds on [Multi-Resolution Points](4784-multi-resolution-points.md) by using pre-shaded textures to represent volumetric density.

## Operating characteristics
**Time Complexity:** $O(N)$ for tree construction and traversal, where $N$ is the number of input points. Rasterization cost is proportional to the number of selected leaf nodes [S1, unofficial].
**Memory Complexity:** $O(N)$ for storing the tree structure [S1, unofficial].
**Performance:** Welsh '02 demonstrated reducing 240k points to 90k points using a frequency-space metric [S1, unofficial].
**Strengths:** Reduces rasterization cost by overlapping areas; reduces storage by retaining only relevant points [S1, unofficial].
**Limitations:** Requires pre-computation of the tree; error metric may not always correlate with visual quality [S1, unofficial].

## Background
Originated from work on volumetric point rendering, specifically by Laur and Hanrahan (Laur '91) who introduced the RMS error metric for point simplification [S1, unofficial]. Welsh '02 applied a frequency-space metric to reduce point counts significantly [S1, unofficial]. The method is used in volume rendering to handle large point clouds efficiently [S1, unofficial].

## Sources
[S1] Volumetric Points - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/teaching/cse616/volPts.pdf, fetched 2026-10-29 (unofficial)
