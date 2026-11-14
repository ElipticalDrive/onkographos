# Hierarchical 3D Gaussian Representations

Also called: Gaussian Pyramid, Multi-resolution Gaussian Pyramid

## Overview
Hierarchical 3D Gaussian Representations render very large datasets in real time by storing a multi-resolution pyramid of Gaussians [S1, unofficial]. This method addresses the memory and performance bottlenecks of standard 3D Gaussian Splatting (3DGS) in city-scale scenes by adapting the representation density to view distance, thereby improving memory efficiency [S1, unofficial]. It sets itself apart from flat 3DGS by organizing primitives into a hierarchy that allows for dynamic level-of-detail (LoD) selection, distinguishing it from sibling methods like [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md) which rely on tile-based training and merging, and [StreetGaussians](6161-streetgaussians.md) which focus on street-level specific priors.

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians (positions, covariances, colors, opacities) representing the scene; camera parameters (position $\mathbf{p}_{\text{cam}}$, orientation, intrinsics).
*   **Output:** A rasterized image.

**Data Structures:**
The method employs a multi-resolution pyramid structure. While specific bit-packing is not detailed in the provided sources, the structure implies a tree or hierarchical organization where nodes at different levels summarize the radiance and opacity of their children [S1, unofficial]. In the specific "Gaussian Hierarchies" variant, this is a recursive tree where non-leaf nodes approximate their children and leaves correspond to original Gaussians [S2, unofficial].

**Steps:**
1.  **Hierarchy Construction:** Organize the Gaussians into a hierarchical structure. In the Gaussian Hierarchies variant, nearby Gaussians are recursively merged into a tree [S2, unofficial].
2.  **Traversal and Cut Selection:** Perform a traversal (e.g., Breadth-First Search) to determine which nodes to render.
3.  **Cut Condition Evaluation:** For each node $i$ and camera $\text{cam}$, evaluate the cut condition $c_{\text{hier}}(i, \text{cam})$ [S2, unofficial].
4.  **Node Selection:**
    *   If $c_{\text{hier}}(i, \text{cam})$ is true, add node $i$ to the cut set and skip its children [S2, unofficial].
    *   If false, continue the traversal to the children of node $i$ [S2, unofficial].
5.  **Rendering:** Rasterize the Gaussians in the cut set. A proper cut set includes no parents or children of any included node, ensuring a valid view-adaptive LoD representation [S2, unofficial].

**Formulas:**
The cut condition for Gaussian Hierarchies is defined as [S2, unofficial]:

$$c_{\text{hier}}(i, \text{cam}) = \left\| \boldsymbol{\mu}_i - \mathbf{p}_{\text{cam}} \right\|_2 \geq m_d(i)$$

Where:
*   $\boldsymbol{\mu}_i$ is the mean (position) of Gaussian node $i$.
*   $\mathbf{p}_{\text{cam}}$ is the camera position.
*   $m_d(i)$ is the distance threshold for node $i$.
*   $m_d(i) = \frac{T}{\max_j s_i^j}$, where $T$ is a global LoD threshold and $s_i^j$ represents the scale of the Gaussian in dimension $j$ [S2, unofficial].

**Parameters:**
*   $T$: Global LoD threshold. Value not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The distance calculation $\left\| \boldsymbol{\mu}_i - \mathbf{p}_{\text{cam}} \right\|_2$ must be robust to floating-point errors near the threshold boundary [G].
*   **Degenerate Input:** If a Gaussian has zero scale ($s_i^j = 0$), $m_d(i)$ becomes undefined. The implementation must handle or prevent zero-scale Gaussians [G].

## Subtopics

### [Budgeted-GS](9644-budgeted-gs.md)
Budgeted-GS is a post-hoc method that converts any trained 3D Gaussian Splatting (3DGS) model into a multi-resolution hierarchy of moment-matched aggregates, enabling real-time rendering of city-scale scenes on consumer GPUs by selecting the level of detail that fits the target device's memory. It solves the problem of excessive memory consumption in large-scale 3DGS by introducing a "capacity floor" derived from optimal transport, which determines the minimum number of primitives required for a given error budget, allowing the same model to serve GPUs with widely different memory capacities. Unlike standard 3DGS which optimizes a fixed set of primitives, Budgeted-GS separates the representation into a hierarchy where a single quality parameter selects the appropriate level of detail per view, ensuring that redundant primitives are merged or dropped based on a pixel-footprint scheduler.

### Gaussian Hierarchies
Gaussian Hierarchies is a recursive tree structure where nearby Gaussians are merged; non-leaf nodes approximate children, and leaves are original Gaussians [S2, unofficial]. It provides a view-adaptive Level of Detail (LoD) representation via a 'cut' set [S2, unofficial]. It is chosen when a strict tree-based traversal with distance-based culling is preferred over other pyramid structures. It uses a cut condition based on camera distance relative to the Gaussian's maximum scale [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time depends on the number of nodes in the cut set. The traversal cost is proportional to the tree size, but culling reduces the number of splats [S2, unofficial].
*   **Memory Complexity:** Significantly reduced compared to flat 3DGS due to the multi-resolution pyramid. Memory usage adapts to view distance [S1, unofficial].
*   **Measured Performance:** Enables real-time rendering of very large datasets [S1, unofficial]. Specific frame rates and resolutions are not given in the sources.
*   **Strengths:** Real-time performance on large scenes; memory efficiency through adaptive LoD [S1, unofficial].
*   **Limitations:** Requires hierarchical construction; performance depends on the efficiency of the cut condition evaluation [S2, unofficial].
*   **When to use:** City-scale or ultra-large scale reconstructions where memory is a constraint [S1, unofficial].
*   **When not to use:** Small-scale scenes where the overhead of hierarchy construction outweighs the benefits.

## Background
The concept of using multi-resolution pyramids for 3D Gaussians was introduced to render very large datasets in real time [S1, unofficial]. The specific "Gaussian Hierarchies" variant was introduced by Kerbl et al. in 2024 [S2, unofficial]. This approach builds on the foundation of 3D Gaussian Splatting, addressing its scalability limitations.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
