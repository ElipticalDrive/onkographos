# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes occlusion relationships for specific viewing regions to avoid rendering primitives that are blocked by others [S1, unofficial]. Unlike static culling methods that rely on local neighbor checks, this approach determines global visibility from a region's perspective, often using occluder fusion to simplify the occlusion geometry [S1, unofficial]. It sets itself apart from its siblings by shifting the computational burden to an offline preprocessing phase to enable faster runtime rendering, though this comes at the cost of significant preprocessing time [S1, unofficial].

## How it works
The method involves generating a visibility map or occlusion data structure for defined regions of the scene. The core challenge is managing the trade-off between preprocessing duration and runtime efficiency [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A 3D model (often represented as triangles or axis-aligned cells) and a set of viewing regions or viewpoints [S1, unofficial].
*   **Output:** Precomputed visibility data indicating which primitives are visible from each region, potentially with occluder fusion applied [S1, unofficial].

**Data Structures:**
Not given in the sources.

**Steps:**
1.  **Region Definition:** Define the viewing regions for which visibility will be precomputed [S1, unofficial].
2.  **Visibility Computation:** For each region, determine which primitives are occluded by others. This may involve occluder fusion to reduce the complexity of the occlusion data [S1, unofficial].
3.  **Storage:** Store the visibility results in a structure accessible at runtime [S1, unofficial].

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Preprocessing Time:** The algorithm can require extremely long preprocessing times. For example, Durand et al.'s algorithm required 33 hours for a model with 6 million triangles [S1, unofficial]. Wonka et al.'s 2.5D algorithm required 9 hours for 8 million triangles [S1, unofficial].
*   **Dataset Limitations:** Early versions were limited to 2.5D datasets [S1, unofficial].

**Pseudocode:**
Not given in the sources.

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It guarantees less than one pixel of screen-space deviation and correct visibility [S1, unofficial]. However, it has high preprocessing costs; for a model with 13 million triangles on a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time is very high, ranging from 9 to 128 hours for models with millions of triangles [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Wonka et al. used 2 processors to overlap visibility computation and rendering at runtime [S1, unofficial].
*   **Strengths:** Enables interactive rendering of large datasets with high accuracy guarantees [S1, unofficial].
*   **Limitations:** Long preprocessing times and limited support for 3D environments in early versions [S1, unofficial].
*   **When to use:** When offline preprocessing time is acceptable and runtime performance for large, complex scenes is critical [S1, unofficial].
*   **When not:** When real-time preprocessing is required or for very large 3D datasets where preprocessing time becomes prohibitive [S1, unofficial].

## Background
The technique was developed by Durand et al. and Wonka et al. [S1, unofficial]. Durand et al. presented a from-region visibility preprocessing algorithm capable of handling 3D environments, contrasting with earlier 2.5D approaches [S1, unofficial]. Wonka et al. presented a similar algorithm with occluder fusion, initially limited to 2.5D datasets [S1, unofficial]. Later work by Wonka et al. explored a from-point approach requiring less preprocessing but was only reported for in-core, 2.5D datasets [S1, unofficial]. Chhugani et al. extended the concept by integrating levels of detail [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
