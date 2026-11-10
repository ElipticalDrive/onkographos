# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-Region Visibility Preprocessing is a technique that precomputes visibility information for specific regions of a scene to optimize rendering performance. It solves the problem of determining which geometry is visible from a given viewpoint or region before the rendering pass, allowing for the culling of occluded or irrelevant geometry. It is distinguished from its siblings by its focus on precomputing visibility for *regions* rather than individual points or static blocks, often involving complex occluder fusion and significant preprocessing overhead.

## How it works
The method involves precomputing visibility data for defined regions of the scene. This data is then used during rendering to determine which parts of the geometry need to be processed.

**Inputs:**
*   Scene geometry (triangles/meshes).
*   Definition of regions (e.g., camera frustums, spatial partitions).

**Outputs:**
*   Visibility maps or masks indicating which geometry is visible from each region.

**Data Structures:**
*   Not explicitly defined in the provided sources, but implied to involve structures storing visibility relationships between regions and geometry.

**Steps:**
1.  **Region Definition:** Define the regions for which visibility will be precomputed.
2.  **Visibility Computation:** For each region, determine which geometry is visible. This may involve ray casting or other visibility algorithms.
3.  **Occluder Fusion:** (Specific to Wonka et al.) Merge occluders to simplify the visibility representation.
4.  **Storage:** Store the computed visibility information for use during rendering.

**Parameters:**
*   Region size and shape.
*   Accuracy requirements.

**Pitfalls:**
*   Long preprocessing times.
*   Limited to specific dataset types (e.g., 2.5D).

## Subtopics

### Wonka et al. Visibility Preprocessing
A from-region visibility preprocessing algorithm with occluder fusion [S1, unofficial]. It used 2 processors to overlap visibility computation and rendering at runtime [S1, unofficial]. It required long preprocessing times (9 hours for a model with 8 million triangles) and was limited to 2.5D datasets [S1, unofficial]. Later work by Wonka et al. used a from-point approach needing little preprocessing, but only reported results for in-core, 2.5D datasets [S1, unofficial].

### Durand et al. Visibility Preprocessing
A from-region visibility preprocessing algorithm that could handle 3D environments, as opposed to 2.5D [S1, unofficial]. However, the algorithm required long preprocessing times (33 hours for a model with 6 million triangles) [S1, unofficial].

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. For a model with 13 million triangles, using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

## Operating characteristics
*   **Preprocessing Time:** Very high. Examples include 9 hours for 8 million triangles (Wonka), 33 hours for 6 million triangles (Durand), and 128 hours for 13 million triangles (Chhugani) [S1, unofficial].
*   **Runtime Performance:** Can overlap visibility computation and rendering [S1, unofficial].
*   **Dataset Limitations:** Early versions were limited to 2.5D datasets [S1, unofficial].
*   **Accuracy:** Can guarantee image accuracy with less than one pixel of screen-space deviation (Chhugani) [S1, unofficial].

## Background
The technique was developed by various researchers including Wonka et al. [152, 153], Durand et al. [41], and Chhugani et al. [20] [S1, unofficial]. It is part of a broader effort to optimize visualization of large datasets, particularly in out-of-core scenarios.

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
