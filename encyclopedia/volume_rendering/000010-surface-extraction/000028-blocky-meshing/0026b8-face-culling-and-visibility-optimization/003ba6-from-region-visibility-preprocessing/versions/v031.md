# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes which geometric primitives are visible from specific regions of the viewing volume, allowing the renderer to discard occluded geometry before the frame is drawn. It solves the problem of expensive per-frame visibility testing for large datasets by shifting the computational load to an offline phase. It differs from its siblings, which typically rely on static voxel occupancy or block-level culling, by explicitly calculating visibility from a defined region of space rather than just local neighbor states [S1, unofficial].

## How it works
The method involves generating a visibility map for a set of predefined regions in the scene. The core input is the scene geometry (typically represented as a mesh or voxelized model) and the set of viewing regions. The output is a data structure mapping each region to a subset of visible primitives or a culling mask.

The algorithm generally follows these steps:
1.  **Region Definition:** The viewing volume is partitioned into regions. In 2.5D implementations, these are often columns or slices [S1, unofficial]. In 3D implementations, these are volumetric blocks [S1, unofficial].
2.  **Occluder Fusion:** To reduce preprocessing time and memory, occluders (geometry that blocks the view) are fused or simplified. This reduces the number of primitives that need to be tested for visibility [S1, unofficial].
3.  **Visibility Computation:** For each region, the algorithm determines which primitives are visible. This involves ray casting or projection-based tests from the region's representative point(s) or volume against the scene geometry.
4.  **Data Structure Construction:** The results are stored in a lookup table or hash map, indexed by region ID, containing the list of visible primitives or a bitmask indicating visibility [S1, unofficial].
5.  **Runtime Rendering:** During rendering, the current camera position is mapped to a region. The precomputed visibility data for that region is retrieved, and only the visible primitives are submitted to the GPU [S1, unofficial].

**Parameters:**
*   **Region Size:** Smaller regions provide higher accuracy but increase preprocessing time and memory usage. [S1, unofficial]
*   **Occluder Fusion Threshold:** Controls the simplification of occluders. [S1, unofficial]

**Pitfalls:**
*   **Preprocessing Time:** The algorithm can require significant preprocessing time, especially for large models. For example, 9 hours for 8 million triangles in 2.5D and 33 hours for 6 million triangles in 3D [S1, unofficial].
*   **Dimensionality Limitations:** Early implementations were limited to 2.5D datasets [S1, unofficial].

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy. It guarantees less than one pixel of screen-space deviation and correct visibility. For a model with 13 million triangles, using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time is high, scaling with the number of triangles and regions. Reported times include 9 hours for 8 million triangles (2.5D) and 33 hours for 6 million triangles (3D) [S1, unofficial].
*   **Memory Complexity:** Depends on the number of regions and the size of the visibility data per region.
*   **Performance:** Enables real-time rendering by reducing the number of triangles processed by the GPU. It overlaps visibility computation and rendering using multiple processors [S1, unofficial].
*   **Strengths:** High accuracy, handles large datasets, reduces runtime visibility testing.
*   **Limitations:** Long preprocessing times, limited to specific dataset types (e.g., 2.5D in early versions), requires significant computational resources for preprocessing [S1, unofficial].

## Background
The concept was introduced by Wonka et al. [152] as a from-region visibility preprocessing algorithm with occluder fusion, limited to 2.5D datasets. Durand et al. [41] extended this to 3D environments. Later work by Wonka et al. [153] explored a from-point approach with less preprocessing. Chhugani et al. [20] developed a system incorporating levels of detail for higher accuracy. These methods contrast with Teller et al.'s algorithm, which also requires long preprocessing but assumes axis-aligned cells [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
