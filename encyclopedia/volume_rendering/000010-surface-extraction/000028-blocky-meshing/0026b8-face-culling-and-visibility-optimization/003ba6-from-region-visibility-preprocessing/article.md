# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes which parts of a 3D scene are visible from specific regions of the camera space, allowing for the culling of occluded geometry during rendering [S1, unofficial]. Unlike parent methods that rely on local voxel adjacency for face culling, this approach handles complex, non-axis-aligned geometry and large-scale datasets by determining visibility from a region rather than a single point [S1, unofficial]. It is distinct from from-point approaches (e.g., [153]) which require little preprocessing but are limited to smaller, in-core datasets [S1, unofficial].

## How it works
The method involves precomputing visibility information for regions of the scene, often utilizing occluder fusion to reduce the number of potential occluders [S1, unofficial]. The specific algorithmic steps for the core visibility computation are not detailed in the sources, but the process involves:

1.  **Region Definition:** The camera space or scene is divided into regions for which visibility will be precomputed [S1, unofficial].
2.  **Occluder Fusion:** Potential occluders are fused to simplify the visibility computation, reducing the complexity of the occlusion test [S1, unofficial].
3.  **Visibility Precomputation:** Visibility data is computed for the defined regions. This step is computationally expensive [S1, unofficial].
4.  **Runtime Rendering:** The precomputed visibility data is used during rendering to cull invisible geometry [S1, unofficial].

**Parameters and Typical Values:**
*   **Preprocessing Time:** Significant. 9 hours for 8 million triangles (Wonka et al.) [S1, unofficial]. 33 hours for 6 million triangles (Durand et al.) [S1, unofficial]. 128 hours for 13 million triangles (Chhugani et al.) [S1, unofficial].
*   **Hardware:** Wonka et al. used 2 processors to overlap computation and rendering [S1, unofficial]. Chhugani et al. used a cluster of 16 PCs [S1, unofficial].
*   **Dataset Size:** Tested on models with 0.6 million to 13 million triangles [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Preprocessing Time:** The primary limitation is the long preprocessing time required, which scales with the number of triangles [S1, unofficial].
*   **Dimensionality:** Early versions (Wonka et al.) were limited to 2.5D datasets [S1, unofficial]. Later versions (Durand et al.) handled 3D environments [S1, unofficial].
*   **Memory:** Wonka et al.'s later from-point approach was limited to in-core datasets, implying memory constraints for larger scenes [S1, unofficial].

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It guarantees less than one pixel of screen-space deviation and correct visibility [S1, unofficial]. For a model with 13 million triangles, using a cluster of 16 PCs, the accuracy guarantee costs 128 hours of precomputation [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time is very high, ranging from 9 to 128 hours for millions of triangles [S1, unofficial].
*   **Memory Complexity:** Not given in the sources. Wonka et al.'s later work was limited to in-core datasets, suggesting high memory requirements [S1, unofficial].
*   **Measured Performance:** Wonka et al. achieved overlap of visibility computation and rendering using 2 processors [S1, unofficial]. Chhugani et al. achieved interactive rendering of large datasets with high accuracy [S1, unofficial].
*   **Strengths:** Handles 3D environments (Durand et al.) [S1, unofficial]; provides high image accuracy with strict visibility guarantees (Chhugani et al.) [S1, unofficial]; overlaps computation and rendering (Wonka et al.) [S1, unofficial].
*   **Limitations:** Very long preprocessing times [S1, unofficial]; limited to 2.5D in early versions (Wonka et al.) [S1, unofficial]; limited to in-core datasets in later from-point variants (Wonka et al.) [S1, unofficial].
*   **When to use:** When high image accuracy and correct visibility are critical, and preprocessing time is acceptable.
*   **When not to use:** When real-time or near-real-time preprocessing is required, or when memory is constrained for very large datasets.

## Background
The concept was introduced by Teller et al., who required long preprocessing times and assumed axis-aligned cells [S1, unofficial]. Wonka et al. [152] presented a from-region visibility preprocessing algorithm with occluder fusion, limited to 2.5D datasets [S1, unofficial]. Durand et al. [41] extended this to 3D environments [S1, unofficial]. Chhugani et al. [20] developed a system focusing on image accuracy and levels of detail [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
