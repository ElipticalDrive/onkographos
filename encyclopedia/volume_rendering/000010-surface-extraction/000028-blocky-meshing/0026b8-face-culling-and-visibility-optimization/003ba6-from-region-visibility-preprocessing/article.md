# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes which parts of a large geometric model are visible from specific regions of the viewing space, allowing the renderer to discard occluded geometry at runtime. It solves the problem of interactive rendering for massive datasets by shifting the computational burden of visibility determination to an offline phase. Unlike immediate culling methods that check neighbors per-voxel, this approach relies on global occlusion analysis, often involving significant preprocessing time to guarantee image accuracy [S1, unofficial].

## How it works
The method involves constructing a visibility structure (such as a visibility polygon or zone) for predefined regions of the scene. This structure identifies "occluders" and "occludees" within the model. During rendering, the system determines which region the camera occupies and uses the precomputed data to filter out triangles that are guaranteed to be hidden by other geometry.

**Inputs and Outputs:**
*   **Input:** A large triangle mesh (often axis-aligned or decomposed into cells) and a partitioning of the viewing space into regions.
*   **Output:** A visibility structure mapping regions to visible subsets of the mesh, and potentially Levels of Detail (LOD) per region.

**Data Structures:**
The specific data structures are not fully detailed in the provided sources, but the algorithm generally requires storing occlusion relationships. For the Chhugani et al. variant, the system stores visibility and LOD information per region [S1, unofficial].

**Steps:**
1.  **Partitioning:** Divide the 3D space or the model into regions. Wonka et al. initially focused on 2.5D datasets, while Durand et al. extended this to full 3D environments [S1, unofficial].
2.  **Visibility Computation:** For each region, compute the set of visible triangles. This involves determining which triangles are occluded by others from any viewpoint within that region.
    *   Wonka et al. used a "from-region" approach with occluder fusion [S1, unofficial].
    *   Durand et al. handled 3D environments, requiring more complex occlusion checks than 2.5D [S1, unofficial].
3.  **LOD Assignment (Optional):** Assign Levels of Detail to visible geometry based on the region and desired accuracy. Chhugani et al. focused on maintaining less than one pixel of screen-space deviation [S1, unofficial].
4.  **Runtime Rendering:** At runtime, determine the camera's current region and render only the precomputed visible subset for that region. Wonka et al. used two processors to overlap visibility computation and rendering [S1, unofficial].

**Parameters:**
*   **Region Size/Granularity:** Not given in the sources. Finer granularity increases preprocessing time but may improve runtime culling efficiency.
*   **Accuracy Guarantee:** Chhugani et al. targeted <1 pixel deviation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Preprocessing Time:** The primary limitation is the extremely long preprocessing time required. Wonka et al. required 9 hours for 8 million triangles [S1, unofficial]. Durand et al. required 33 hours for 6 million triangles [S1, unofficial]. Chhugani et al. required 128 hours for 13 million triangles on a cluster of 16 PCs [S1, unofficial].
*   **Dataset Limitations:** Early versions were limited to 2.5D datasets [S1, unofficial]. Later work by Wonka et al. attempted a from-point approach with less preprocessing but still reported only on 2.5D in-core datasets [S1, unofficial].
*   **Memory:** The sources do not specify memory requirements, but storing visibility for millions of triangles across many regions is likely substantial.

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy. It guarantees less than one pixel of screen-space deviation and correct visibility. However, it is computationally expensive, requiring 128 hours of precomputation on a 16-PC cluster for a 13 million triangle model [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Preprocessing time is very high, scaling poorly with model size. Reported times range from 9 hours (8M triangles) to 128 hours (13M triangles) [S1, unofficial].
*   **Memory Complexity:** Not given in the sources.
*   **Performance:** Enables interactive rendering of large datasets by reducing the number of triangles sent to the GPU at runtime.
*   **Strengths:** Guarantees visibility correctness and image accuracy.
*   **Limitations:** Prohibitive preprocessing times; limited applicability to 2.5D in early versions; requires significant computational resources for preprocessing.
*   **When to use:** When interactive frame rates are critical for very large static datasets and offline preprocessing time is acceptable.
*   **When not to use:** For dynamic scenes or when preprocessing time is a bottleneck.

## Background
From-region visibility preprocessing was developed in the context of out-of-core visualization of large datasets. Key contributors include Teller et al. (who required long preprocessing and assumed axis-aligned cells), Wonka et al. (who introduced occluder fusion and 2.5D support), Durand et al. (who extended to 3D), and Chhugani et al. (who integrated LOD). The approach contrasts with PLP and cPLP, which require little preprocessing [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
