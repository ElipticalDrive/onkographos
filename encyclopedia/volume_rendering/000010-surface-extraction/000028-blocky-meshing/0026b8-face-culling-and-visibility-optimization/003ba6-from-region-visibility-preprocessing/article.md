# From-Region Visibility Preprocessing

Also called: Durand et al. Visibility Preprocessing, Wonka et al. Visibility Preprocessing

## Overview
From-region visibility preprocessing is a technique that precomputes which parts of a 3D scene are visible from specific viewpoints or regions, allowing for the culling of occluded geometry during rendering [S1, unofficial]. It differs from standard face culling by operating on larger geometric clusters or regions rather than individual voxel faces, aiming to reduce the number of triangles submitted to the graphics pipeline [S1, unofficial]. Unlike [Face Culling](8735-face-culling.md) which relies on local neighbor checks, this method often involves global or semi-global visibility analysis, which can incur significant preprocessing costs [S1, unofficial].

## How it works
The method involves constructing a visibility structure (such as a visibility polygon or zone) for a given region or viewpoint. The core logic typically involves:
1.  **Region Definition**: Defining a set of viewpoints or a spatial region from which visibility is queried [S1, unofficial].
2.  **Occluder Fusion**: Identifying occluding geometry and merging it to simplify the visibility computation [S1, unofficial].
3.  **Visibility Computation**: Determining which triangles or clusters are visible from the defined region. This may involve ray casting or projection-based techniques [S1, unofficial].
4.  **Preprocessing**: Storing the visibility information for fast runtime lookup [S1, unofficial].

**Pitfalls**:
*   **Preprocessing Time**: The algorithm can require extremely long preprocessing times, making it impractical for dynamic scenes or very large models without significant hardware resources [S1, unofficial].
*   **Dataset Limitations**: Early implementations were limited to 2.5D datasets, restricting their applicability to true 3D environments [S1, unofficial].

## Subtopics

### Chhugani et al. Visibility and LOD System
A system that precomputes from-region visibility and levels of detail per region, focusing on image accuracy [S1, unofficial]. It guarantees less than one pixel of screen-space deviation and correct visibility [S1, unofficial]. However, it requires substantial computational resources; for a model with 13 million triangles using a cluster of 16 PCs, the precomputation takes 128 hours [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Preprocessing time is very high. For example, 9 hours for 8 million triangles [S1, unofficial] and 33 hours for 6 million triangles [S1, unofficial].
*   **Memory Complexity**: Not given in the sources.
*   **Strengths**: Can handle large datasets with interactive rendering speeds after preprocessing [S1, unofficial].
*   **Limitations**: Long preprocessing times, limited to 2.5D in some implementations, and requires significant computational clusters for large models [S1, unofficial].
*   **When to use**: Static, very large datasets where preprocessing time is acceptable and interactive rendering of occluded geometry is critical [S1, unofficial].

## Background
The concept was introduced by Wonka et al. [152] as a from-region visibility preprocessing algorithm with occluder fusion, which used 2 processors to overlap visibility computation and rendering [S1, unofficial]. Durand et al. [41] extended this to handle 3D environments, though with even longer preprocessing times (33 hours for 6 million triangles) [S1, unofficial]. Later work by Wonka et al. [153] shifted to a from-point approach with less preprocessing but remained limited to 2.5D datasets [S1, unofficial].

## Sources
[S1] New Techniques for Out-Of-Core Visualization of Large Datasets, https://ctsilva.github.io/thesis/wagner-correa-thesis.pdf, fetched 2026-10-30 (unofficial)
