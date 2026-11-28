# Hierarchical 3D Gaussian Representations

Also called: Gaussian Pyramid, Multi-resolution Gaussian Pyramid

## Overview
Hierarchical 3D Gaussian Representations render very large datasets in real time by storing a multi-resolution pyramid of Gaussians [S1, unofficial]. This method addresses memory and bandwidth bottlenecks in flat 3D Gaussian Splatting by organizing primitives into a spatial hierarchy that adapts to view distance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$ representing the scene [S1, unofficial].
*   **Output:** A multi-resolution pyramid of Gaussians enabling real-time rendering of large-scale scenes [S1, unofficial].

**Data Structures:**
The method stores a multi-resolution pyramid of Gaussians [S1, unofficial]. The exact bit-packing and field layouts for the aggregated Gaussians are not given in the sources [S1, unofficial].

**Steps:**
1.  **Hierarchy Construction:** Build a multi-resolution pyramid over the scene geometry [S1, unofficial]. The exact construction algorithm is not given in the sources [S1, unofficial].
2.  **Aggregation:** Merge child Gaussians into parent nodes [S1, unofficial]. The exact aggregation formulas are not given in the sources [S1, unofficial].
3.  **Traversal and Selection:** Traverse the hierarchy during rendering to select the appropriate level of detail for each view [S1, unofficial]. The exact traversal algorithm is not given in the sources [S1, unofficial].
4.  **Splatting:** Render the selected Gaussians using standard 3DGS splatting techniques [S1, unofficial].

**Formulas:**
Not given in the sources [S1, unofficial].

**Parameters:**
*   **Hierarchy Depth:** Determines the number of resolution levels [S1, unofficial].
*   **Aggregation Method:** How child Gaussians are merged into parent nodes [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Not given in the sources [S1, unofficial].
*   **Boundaries:** The method handles boundaries by storing Gaussians only in occupied regions [S1, unofficial].

## Subtopics

### [Budgeted-GS](9644-budgeted-gs.md)
A post-hoc method that converts any trained 3DGS model into a multi-resolution hierarchy of moment-matched aggregates, enabling real-time rendering on consumer GPUs by selecting the LoD that fits the target device's memory. It uses a "capacity floor" derived from optimal transport.

### [Gaussian Hierarchies](10313-gaussian-hierarchies.md)
Enables real-time rendering of ultra-large-scale datasets by organizing Gaussians into a recursive tree structure. It differs from flat splatting by introducing spatial hierarchy and from Budgeted-GS by relying on a strict tree-based cut condition.

## Operating characteristics
*   **Time Complexity:** Real-time rendering is achieved through adaptive level selection and efficient GPU execution [S1, unofficial]. The exact complexity formula is not given in the sources [S1, unofficial].
*   **Memory Complexity:** Memory efficiency is improved by storing Gaussians only in occupied regions and using a multi-resolution pyramid [S1, unofficial]. The exact scaling formula is not given in the sources [S1, unofficial].
*   **Strengths:** Real-time performance on large datasets; adaptive detail selection; memory efficiency [S1, unofficial].
*   **Limitations:** The specific implementation details (e.g., exact aggregation formulas, traversal algorithms) are not provided in the sources [S1, unofficial].
*   **When to use:** When rendering very large datasets in real time is required [S1, unofficial].
*   **When not to use:** Not given in the sources [S1, unofficial].

## Background
The concept of hierarchical 3D Gaussian representations is part of the broader effort to scale 3D Gaussian Splatting to city-scale scenes [S1, unofficial]. It builds on [CityGaussian / VastGaussian / DoGaussian](6143-citygaussian-vastgaussian-dogaussian.md), which divides the city into tiles and uses a LoD pyramid for streaming [S1, unofficial]. Other related works include Octree-GS [142], LODGE [144], HiGS [145], and FilterGS [146], which propose various hierarchical or filtering structures for efficient rendering [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
