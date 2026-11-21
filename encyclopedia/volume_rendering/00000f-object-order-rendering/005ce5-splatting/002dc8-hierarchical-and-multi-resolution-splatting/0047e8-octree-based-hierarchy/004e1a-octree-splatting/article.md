# Octree Splatting

Also called: Laur-Hanrahan Octree Splatting

## Overview
Octree Splatting is a hierarchical object-order volume rendering technique that renders non-leaf octree nodes as single primitives to accelerate the visualization of dense volumetric data [S1, unofficial][S2, book]. It distinguishes itself from standard splatting by actively utilizing internal tree nodes as renderable kernels, rather than restricting them to occlusion culling, thereby reducing the number of draw calls and traversal overhead for homogeneous regions [S1, unofficial][S2, book].

## How it works
**Inputs:** A volume dataset organized into an octree structure.
**Outputs:** A 2D image composed of splatted primitives.

**Data Structures:**
Each octree node stores:
1.  **Spatial Extent:** Implicitly defined by the octree hierarchy level and position.
2.  **Children's Average ($\mu$):** The weighted average of the scalar values (e.g., density or opacity) of all leaf nodes in the subtree.
3.  **Error Metric ($\sigma$):** A metric based on the standard deviation of the children's values relative to the average $\mu$ [S1, unofficial][S2, book].

**Algorithm Steps:**
1.  **Tree Construction:**
    *   Build the octree from the input volume.
    *   For each internal node, compute $\mu$ as the average of its children's values.
    *   Compute the error metric $\sigma$ based on the standard deviation of the children's values [S1, unofficial][S2, book].

2.  **Rendering (Node Selection):**
    *   Traverse the octree from the root.
    *   For each node, compare its pre-computed error metric $\sigma$ against a user-defined threshold $\epsilon$ [S1, unofficial][S2, book].
    *   **Selection Criterion:** If $\sigma \leq \epsilon$, the node is selected as a leaf for rendering. Otherwise, recurse to its children [S1, unofficial][S2, book].

3.  **Primitive Approximation:**
    *   When a non-leaf node is selected, it is approximated by a single kernel (splat) [S1, unofficial][S2, book].
    *   **Footprint Radius:** The radius $R$ of the splat is set to twice the radius of the kernels of its immediate children ($R_{child}$). This creates a magnified footprint that covers the spatial extent of the children [S1, unofficial][S2, book].
    *   **Kernel Value:** The intensity or opacity of the splat is determined by the node's stored average $\mu$ [S1, unofficial][S2, book].

4.  **Compositing:**
    *   Render the selected splats using standard alpha blending or accumulation techniques. The philosophy is that the observable error from using a coarser node is small when weighted by the transparency of the volume [S1, unofficial].

**Parameters:**
*   $\epsilon$: Pre-set error threshold for node selection. Typical values depend on the specific volume and desired visual fidelity; not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The error metric relies on the statistical distribution of values within the subtree. If the distribution is highly non-uniform, the standard deviation may not accurately reflect visual error [S1, unofficial].
*   **Boundary:** The magnified footprint ($2 \times R_{child}$) may extend beyond the actual data boundaries of the children, potentially causing artifacts if not clamped or handled correctly during compositing [S1, unofficial][S2, book].

## Subtopics

## Operating characteristics
*   **Time Complexity:** Rendering time is proportional to the number of selected leaf nodes in the hierarchy. The traversal is $O(N)$ in the worst case (all leaves), but typically much faster for smooth volumes due to early termination at internal nodes.
*   **Memory Complexity:** Requires storing the octree structure plus the average and error metric for each node. This is generally higher than a raw voxel grid but allows for significant reduction in rendering primitives.
*   **Strengths:** Efficient culling of homogeneous regions; reduced number of draw calls; simple implementation based on standard splatting.
*   **Limitations:** The fixed magnification factor (2x) may lead to over-blurring in some cases; the error metric is static and view-independent (unlike adaptive LOD methods) [S1, unofficial][S2, book].
*   **When to use:** Suitable for dense, smooth volumetric data where visual fidelity can be approximated by statistical averages.
*   **When not to use:** Unsuitable for data with sharp edges or high-frequency details where the standard deviation error metric fails to capture local variations.

## Background
The method was proposed by Laur and Hanrahan [S1, unofficial][S2, book]. It is based on the idea that non-leaf nodes in an octree can be approximated by kernels with a radius twice that of their children, providing a magnified footprint that represents the aggregate data of the subtree [S1, unofficial][S2, book]. This approach contrasts with other hierarchical methods that use internal nodes only for occlusion culling [S1, unofficial][S2, book].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
