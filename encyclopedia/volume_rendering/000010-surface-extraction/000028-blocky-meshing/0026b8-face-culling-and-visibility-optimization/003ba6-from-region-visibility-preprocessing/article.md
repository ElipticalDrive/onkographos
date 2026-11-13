# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes which geometric primitives are visible from specific regions of a scene to optimize rendering performance. It solves the problem of redundant rendering of occluded geometry by determining visibility sets for spatial regions in advance. It differs from sibling methods like [Face Culling](8735-face-culling.md) which operate on local voxel adjacency, by operating on global or semi-global geometric occlusion relationships [S1, unofficial].

## How it works
The method involves precomputing visibility information for regions of the scene, often involving occluder fusion to reduce the complexity of the occlusion test.

1.  **Input**: A 3D scene model (typically represented as a mesh of triangles) and a partitioning of the scene into regions or viewpoints.
2.  **Preprocessing**:
    *   For each region, determine the set of triangles visible from that region.
    *   Apply occluder fusion to simplify the set of occluding geometry [S1, unofficial].
    *   Store the visibility data (e.g., which triangles are visible from which regions).
3.  **Runtime**:
    *   Determine the current viewer's region.
    *   Retrieve the precomputed visibility set for that region.
    *   Render only the triangles in the visible set [S1, unofficial].

**Parameters**:
*   **Region Definition**: How the scene is partitioned (e.g., grid cells, hierarchical structures). Not given in the sources.
*   **Occluder Fusion Threshold**: Criteria for merging occluders. Not given in the sources.

**Edge Cases**:
*   **Large Models**: Preprocessing time scales significantly with model complexity. For example, 9 hours for 8 million triangles [S1, unofficial].
*   **Dimensionality**: Early implementations were limited to 2.5D datasets [S1, unofficial].

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It guarantees less than one pixel of screen-space deviation and correct visibility. For a model with 13 million triangles, using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

## Operating characteristics
*   **Preprocessing Time**: Very high. Reported times include 9 hours for 8 million triangles [S1, unofficial], 33 hours for 6 million triangles [S1, unofficial], and 128 hours for 13 million triangles on a 16-PC cluster [S1, unofficial].
*   **Runtime Performance**: Enables interactive rendering by reducing the number of triangles to be processed per frame [S1, unofficial].
*   **Memory**: Requires storage for visibility data structures. Not given in the sources.
*   **Limitations**: Long preprocessing times limit applicability to very large models. Early versions were limited to 2.5D datasets [S1, unofficial].

## Background
The concept was introduced by Wonka et al. [152] as a from-region visibility preprocessing algorithm with occluder fusion, limited to 2.5D datasets [S1, unofficial]. Durand et al. [41] extended this to handle 3D environments, though with significant preprocessing costs (33 hours for 6 million triangles) [S1, unofficial]. Later work by Wonka et al. [153] shifted to a from-point approach with less preprocessing but remained limited to in-core, 2.5D datasets [S1, unofficial]. Chhugani et al. [20] developed a system integrating visibility and LOD per region for high image accuracy [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
