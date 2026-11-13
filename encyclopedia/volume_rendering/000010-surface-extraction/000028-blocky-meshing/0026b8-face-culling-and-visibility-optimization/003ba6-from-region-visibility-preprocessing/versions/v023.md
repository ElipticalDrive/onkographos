# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes the visibility of geometric primitives (such as triangles) from specific regions of the viewing space to optimize rendering performance [S1, unofficial]. It solves the problem of expensive per-frame visibility determination for large datasets by shifting the computational load to an offline or semi-offline phase [S1, unofficial]. Unlike parent methods that rely on voxel occupancy for immediate face culling, this approach handles complex, non-axis-aligned geometry and provides guarantees on image accuracy, though it incurs significant preprocessing overhead [S1, unofficial].

## How it works
The method involves constructing a visibility structure that maps regions of the camera space to subsets of the scene geometry. The core logic relies on occluder fusion and conservative visibility computation to ensure that no visible detail is incorrectly culled [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A large-scale 3D model (represented as a mesh of triangles) and a partitioning of the viewing volume into regions.
*   **Output:** A precomputed visibility map associating each region with a subset of visible triangles, potentially with associated Levels of Detail (LOD) [S1, unofficial].

**Data Structures:**
The specific internal data structures are not detailed in the sources, but the system maintains a mapping from spatial regions to triangle subsets [S1, unofficial].

**Steps:**
1.  **Region Partitioning:** The viewing volume is divided into distinct regions [S1, unofficial].
2.  **Visibility Computation:** For each region, the algorithm determines which triangles are potentially visible. This step often employs occluder fusion to reduce the number of occluders considered [S1, unofficial].
3.  **LOD Assignment:** Levels of detail are precomputed for each region to balance accuracy and performance [S1, unofficial].
4.  **Runtime Rendering:** At runtime, the system identifies the current camera region and retrieves the precomputed triangle subset [S1, unofficial].

**Parameters:**
*   **Preprocessing Time:** Highly dependent on model complexity. For example, 9 hours for 8 million triangles [S1, unofficial].
*   **Hardware:** Can utilize multiple processors to overlap computation and rendering [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Preprocessing Cost:** The primary pitfall is the extremely long preprocessing time required for large models [S1, unofficial].
*   **Dimensionality Limitations:** Early implementations were limited to 2.5D datasets [S1, unofficial].

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It guarantees less than one pixel of screen-space deviation and correct visibility [S1, unofficial]. For a model with 13 million triangles, using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time is substantial, scaling with model complexity. Reported times include 9 hours for 8 million triangles [S1, unofficial] and 33 hours for 6 million triangles [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Allows for interactive rendering of large datasets with high accuracy [S1, unofficial].
*   **Strengths:** Handles 3D environments (unlike some 2.5D predecessors) and provides strict accuracy guarantees [S1, unofficial].
*   **Limitations:** Long preprocessing times make it unsuitable for dynamic scenes or rapid prototyping [S1, unofficial]. Limited to in-core datasets in some reported results [S1, unofficial].

## Background
The concept was introduced by Wonka et al. [152] as a from-region visibility preprocessing algorithm with occluder fusion, which used 2 processors to overlap visibility computation and rendering at runtime [S1, unofficial]. Durand et al. [41] extended this to handle 3D environments, though it required 33 hours of preprocessing for a 6 million triangle model [S1, unofficial]. Chhugani et al. [20] later developed a system focusing on image accuracy and LOD, costing 128 hours for 13 million triangles [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
