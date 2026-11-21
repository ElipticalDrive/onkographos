# Hierarchical 3D Gaussian Representations

Also called: Gaussian Pyramid, Multi-resolution Gaussian Pyramid

## Overview
Hierarchical 3D Gaussian Representations render very large datasets in real time by storing a multi-resolution pyramid of Gaussians [S1, unofficial]. This method addresses the memory and bandwidth bottlenecks inherent in flat 3D Gaussian Splatting (3DGS) by organizing primitives into a spatial hierarchy, allowing the renderer to adaptively select detail levels based on view distance and available resources [S1, unofficial]. It sets itself apart from flat splatting by introducing spatial locality and from sibling methods like Budgeted-GS by relying on strict tree-based cut conditions rather than global budget optimization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$ representing the scene, typically derived from a trained 3DGS model.
*   **Output:** A hierarchical data structure (e.g., octree or pyramid) where each node contains aggregated Gaussian parameters, enabling real-time rendering of large-scale scenes [S1, unofficial].

**Data Structures:**
The method constructs a multi-resolution pyramid or tree structure. Each node in the hierarchy summarizes the radiance and opacity of its children, effectively merging Gaussians in occupied regions [S1, unofficial]. The exact bit-packing and field layouts for the aggregated Gaussians are not specified in the provided sources.

**Steps:**
1.  **Hierarchy Construction:** Build a spatial hierarchy (such as an octree) over the scene geometry. Each node is associated with a merged Gaussian that summarizes the radiance and opacity of its children [S1, unofficial].
2.  **Aggregation:** For each node, aggregate the properties of its child Gaussians. This creates a coarser representation at higher levels of the pyramid [S1, unofficial].
3.  **Traversal and Selection:** During rendering, traverse the hierarchy to select the appropriate level of detail (LoD) for each view. The selection is view-adaptive, allowing the renderer to choose a sparse subset of nodes that approximate the scene geometry at varying distances [S1, unofficial].
4.  **Splatting:** Render the selected Gaussians using standard 3DGS splatting techniques. Gaussians are only stored in occupied regions, which naturally adapts to view distance and improves memory efficiency [S1, unofficial].

**Parameters:**
*   **Hierarchy Depth:** Determines the number of resolution levels. Not given in the sources.
*   **Aggregation Method:** How child Gaussians are merged into parent nodes. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Aggregating Gaussians may introduce approximation errors. The sources do not specify error bounds.
*   **Boundaries:** The method handles boundaries by only storing Gaussians in occupied regions [S1, unofficial].

## Subtopics

### [Budgeted-GS](9644-budgeted-gs.md)
A post-hoc method that converts any trained 3DGS model into a multi-resolution hierarchy of moment-matched aggregates, enabling real-time rendering on consumer GPUs by selecting the LoD that fits the target device's memory. It uses a "capacity floor" derived from optimal transport.

### [Gaussian Hierarchies](10313-gaussian-hierarchies.md)
Enables real-time rendering of ultra-large-scale datasets by organizing Gaussians into a recursive tree structure. It differs from flat splatting by introducing spatial hierarchy and from Budgeted-GS by relying on a strict tree-based cut condition.

## Operating characteristics
*   **Time Complexity:** Real-time rendering is achieved through adaptive level selection and efficient GPU execution [S1, unofficial]. The exact complexity is not specified.
*   **Memory Complexity:** Memory efficiency is improved by storing Gaussians only in occupied regions and using a multi-resolution pyramid [S1, unofficial].
*   **Strengths:** Real-time performance on large datasets; adaptive detail selection; memory efficiency.
*   **Limitations:** The specific implementation details (e.g., exact aggregation formulas, traversal algorithms) are not provided in the sources.
*   **When to use:** When rendering very large datasets in real time is required.
*   **When not to use:** When high-fidelity, non-hierarchical rendering is needed without memory constraints.

## Background
The concept of hierarchical 3D Gaussian representations is part of the broader effort to scale 3D Gaussian Splatting to city-scale scenes. It builds on [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md), which divides the city into tiles and uses a LoD pyramid for streaming. Other related works include Octree-GS [142], LODGE [144], HiGS [145], and FilterGS [146], which propose various hierarchical or filtering structures for efficient rendering [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
