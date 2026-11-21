# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique for optimizing the rendering of large-scale 3D environments by precomputing which geometric primitives are visible from specific regions of the scene. It solves the problem of expensive per-frame visibility culling for massive datasets by shifting the computational burden to an offline phase. It is distinguished from its siblings, such as [Face Culling and Visibility Optimization](3861-face-culling-and-visibility-optimization.md), which rely on local voxel topology and axis-aligned assumptions, by operating on general geometry (triangles) and using occluder fusion to reduce the number of visible primitives, albeit at the cost of significantly longer preprocessing times [S1, S2, S3, S4, S5].

## How it works
The method involves offline preprocessing to generate visibility data structures that map viewing regions to subsets of the scene geometry.

**Inputs:**
*   A 3D scene model defined by a set of triangles (e.g., millions of triangles).
*   A partitioning of the scene into "regions" or viewpoints.

**Outputs:**
*   A visibility map associating each region with a subset of visible triangles.
*   Optional: Levels of Detail (LOD) information per region.

**Data Structures:**
*   The specific internal layout of the visibility data structures is not detailed in the sources. The output is generally a lookup table or map used during runtime to cull triangles not visible from the current camera position [S1, S2, S3, S4, S5].

**Steps:**
1.  **Region Definition:** The 3D space is divided into regions. The specific method of partitioning (e.g., grid, hierarchical) is not specified in the provided notes, but the goal is to cover the walkthrough or viewing volume [S1, S2, S3, S4, S5].
2.  **Visibility Computation:** For each region, the algorithm determines which triangles are visible.
    *   Wonka et al. [S2] used a from-region approach with **occluder fusion**. This likely involves grouping occluders to reduce the number of visibility tests.
    *   Durand et al. [S3] extended this to full 3D environments, unlike earlier 2.5D limitations.
3.  **Preprocessing:** The visibility data is computed offline. This step is computationally intensive.
4.  **Runtime Rendering:** During rendering, the system identifies the current region and retrieves the precomputed visible triangle subset.
    *   Wonka et al. [S2] used 2 processors to overlap visibility computation and rendering at runtime, suggesting a streaming or incremental update mechanism for the visibility data [S2].

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Preprocessing Time:** The method requires very long preprocessing times. For example, 9 hours for 8 million triangles (Wonka et al.) and 33 hours for 6 million triangles (Durand et al.) [S2, S3].
*   **Dataset Limitations:** Early versions (Wonka et al. [S2]) were limited to 2.5D datasets. Later work [S2] moved to a from-point approach for in-core, 2.5D datasets, but the full 3D from-region method remains heavy [S2, S3].
*   **Scalability:** The preprocessing cost scales poorly with triangle count. Chhugani et al. [S5] reported 128 hours for 13 million triangles on a cluster of 16 PCs [S5].

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S5]. It guarantees less than one pixel of screen-space deviation and correct visibility [S5]. For a model with 13 million triangles, using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S5].

## Operating characteristics
*   **Time Complexity:** Preprocessing time is very high, ranging from 9 to 128 hours for models with 6 to 13 million triangles [S2, S3, S5].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Wonka et al. [S2] achieved interactive rendering by overlapping visibility computation and rendering using 2 processors. Chhugani et al. [S5] achieved interactive rendering of large datasets with high image accuracy.
*   **Strengths:** High image accuracy with correct visibility; handles large datasets interactively at runtime; supports LOD.
*   **Limitations:** Extremely long preprocessing times; limited to 2.5D in some early implementations; requires significant computational resources for preprocessing (e.g., clusters of PCs) [S2, S3, S5].
*   **When to use:** When interactive frame rates are required for very large, complex scenes and offline preprocessing time is acceptable.
*   **When not:** When rapid scene updates or dynamic geometry are required, or when preprocessing resources are limited.

## Background
The method originated from research into out-of-core visualization of large datasets. Wonka et al. [S2] presented a from-region visibility preprocessing algorithm with occluder fusion, initially limited to 2.5D datasets. Durand et al. [S3] extended the approach to handle full 3D environments. Chhugani et al. [S5] developed a system incorporating LODs to ensure image accuracy. These methods contrast with earlier approaches like Teller et al., which also required long preprocessing but assumed axis-aligned cells, and later approaches like PLP/cPLP which required little preprocessing but made no geometry assumptions [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
