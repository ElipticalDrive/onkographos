# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes which geometric primitives are visible from specific regions of a scene to optimize rendering performance [S1, unofficial]. It solves the problem of expensive per-frame visibility checks in large datasets by shifting the computational burden to an offline or semi-offline phase [S1, unofficial]. Unlike sibling methods that rely on simple voxel neighbor checks or block placement, this approach handles complex, non-axis-aligned geometry (in later variants) and focuses on occluder fusion to reduce the number of visible triangles [S1, unofficial].

## How it works
The method involves constructing a visibility map for predefined regions in the scene space. The core challenge is the high computational cost of this preprocessing step, particularly when dealing with large triangle counts [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A 3D scene model (represented as a mesh of triangles) and a partitioning of the view volume into regions.
*   **Output:** A visibility table or map associating each region with a subset of visible triangles, potentially with occluder fusion applied [S1, unofficial].

**Data Structures:**
*   **Region Grid:** A spatial partitioning of the environment. In early 2.5D variants, this was limited to planar slices [S1, unofficial]. In 3D variants, it covers volumetric regions [S1, unofficial].
*   **Visibility List:** For each region, a list of triangle indices that are not fully occluded by other geometry when viewed from that region [S1, unofficial].

**Steps:**
1.  **Scene Partitioning:** Divide the scene space into regions. The granularity affects accuracy and preprocessing time [S1, unofficial].
2.  **Visibility Computation:** For each region, determine which triangles are visible. This involves ray casting or projection-based tests against the scene geometry [S1, unofficial].
3.  **Occluder Fusion:** Merge occluders to simplify the visibility representation. If a triangle is occluded by a larger, closer triangle, it may be removed from the visible list for that region [S1, unofficial].
4.  **Storage:** Store the resulting visibility maps. At runtime, the system queries the map for the current camera position to determine which triangles to render [S1, unofficial].

**Parameters:**
*   **Region Size:** Smaller regions yield higher accuracy but increase preprocessing time and memory [S1, unofficial].
*   **Accuracy Guarantee:** Some systems enforce a maximum screen-space deviation (e.g., less than one pixel) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Preprocessing Time:** The primary pitfall is the extremely long preprocessing time required for large models. For example, 9 hours for 8 million triangles in 2.5D, and 33 hours for 6 million triangles in 3D [S1, unofficial].
*   **Dataset Limitations:** Early versions were limited to 2.5D datasets [S1, unofficial].
*   **Memory:** Storing visibility data for large scenes can be memory-intensive [S1, unofficial].

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It guarantees less than one pixel of screen-space deviation and correct visibility [S1, unofficial]. However, it is computationally expensive: for a model with 13 million triangles on a cluster of 16 PCs, it requires 128 hours of precomputation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time is very high, scaling poorly with triangle count. Reported times include 9 hours for 8 million triangles (2.5D) and 33 hours for 6 million triangles (3D) [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Allows for fast runtime rendering by reducing the number of triangles to process, but requires significant offline computation [S1, unofficial].
*   **Strengths:** Handles complex geometry (in 3D variants) and provides accurate visibility culling [S1, unofficial].
*   **Limitations:** Extremely long preprocessing times; early versions limited to 2.5D datasets [S1, unofficial].
*   **When to use:** When offline preprocessing time is acceptable and runtime performance for large, complex scenes is critical.
*   **When not to use:** For dynamic scenes where geometry changes frequently, or when rapid iteration is needed.

## Background
The concept was introduced by Wonka et al. [152] as a from-region visibility preprocessing algorithm with occluder fusion, initially limited to 2.5D datasets [S1, unofficial]. Durand et al. [41] extended this to handle 3D environments, though at the cost of even longer preprocessing times (33 hours for 6 million triangles) [S1, unofficial]. Later work by Wonka et al. [153] shifted to a from-point approach with less preprocessing but remained limited to in-core, 2.5D datasets [S1, unofficial]. Chhugani et al. [20] further developed the system to include levels of detail and accuracy guarantees [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
