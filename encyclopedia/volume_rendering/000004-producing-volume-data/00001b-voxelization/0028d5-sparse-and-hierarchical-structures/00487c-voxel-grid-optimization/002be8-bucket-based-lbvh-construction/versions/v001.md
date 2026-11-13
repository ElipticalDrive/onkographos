# Bucket-based LBVH construction

Also called: Fernandes and Walter LBVH construction

## Overview
Bucket-based LBVH construction groups voxels into variable-sized buckets rather than using a uniform grid with a fixed brick size [S1, unofficial]. This approach is designed to handle sparse volumes more efficiently than fixed-grid methods, allowing for two to three times higher frame rates during ray marching by skipping empty space [S1, unofficial]. It sets itself apart from siblings like [Bounding Volume Optimization](3583-bounding-volume-optimization.md) by dynamically determining bucket boundaries based on voxel distribution rather than static spatial subdivisions.

## How it works
**Inputs and Outputs:**
*   **Input:** A Depth-Sorted Voxels Texture containing all selected voxels, sorted by depth.
*   **Output:** A Buckets Texture storing the position (index) of the last voxel (boundary voxel) of each bucket.

**Data Structures:**
1.  **Depth-Sorted Voxels Texture:** A texture where each entry corresponds to a voxel. The alpha channel stores the voxel's index in the sorted list.
2.  **Auxiliary Texture:** A temporary texture used to identify bucket boundaries.
    *   **R/G Channels:** Store the bucket coordinates (indices) in the final Buckets Texture.
    *   **B Channel:** Stores a "false" z-value. If the voxel is a bucket boundary, the value is inside the viewing frustum (e.g., 0.5). If not, it is outside the frustum.
    *   **Alpha Channel:** Stores the voxel index from the Depth-Sorted Voxels Texture.
3.  **Buckets Texture:** The final output texture. Each entry corresponds to a bucket and stores the index of the last voxel in that bucket within the Depth-Sorted Voxels Texture.

**Steps:**
The computation is split into two steps to avoid inefficient texture fetches in vertex shaders [S2, unofficial].

1.  **Identify Boundary Voxels (Fragment Shader Pass):**
    *   Render the Depth-Sorted Voxels Texture into the Auxiliary Texture using a fragment shader.
    *   For each voxel, determine if it is a bucket boundary. A voxel $i$ is a boundary if the center of the next voxel ($i+1$) falls inside the next bucket slab [S2, unofficial].
    *   Write the bucket coordinates to R/G.
    *   Write the false z-value (inside frustum for boundary, outside for non-boundary) to B.
    *   Write the voxel index to Alpha.

2.  **Construct Buckets Texture (Vertex Shader Pass):**
    *   Use the Auxiliary Texture as a Vertex Buffer Object (VBO).
    *   Render using `glPoints` of size one.
    *   The graphics pipeline performs an early-depth test. Points with z-values outside the frustum (non-boundary voxels) are rejected.
    *   Only points corresponding to bucket boundary voxels (z-value inside frustum) proceed to the fragment shader.
    *   The fragment shader writes the voxel index (from the Alpha channel of the VBO) into the Buckets Texture at the location specified by the bucket coordinates (R/G).

**Pitfalls:**
*   Determining if a voxel is a bucket boundary requires comparing the current voxel's bucket with the next voxel's bucket. This is expensive in vertex shaders due to inefficient texture fetches [S2, unofficial]. The two-pass approach mitigates this by moving the comparison logic to the fragment shader in the first pass.

## Subtopics

### Bucket Size Computation
This stage computes the Buckets Texture on the GPU. It cannot be done directly from the Depth-Sorted Voxels Texture using fragment shaders because there is no direct relationship between the texture containing all voxels and the texture containing one voxel per bucket [S2, unofficial]. The method relies on the two-step rendering pipeline described in the main algorithm to efficiently identify and store bucket boundaries.

## Operating characteristics
*   **Performance:** Allows for two to three times higher frame rates than using no empty-space skipping during ray marching [S1, unofficial].
*   **Hardware:** Designed for GPU implementation, leveraging fragment and vertex shaders.
*   **Strengths:** Efficient for sparse volumes; avoids fixed brick size limitations.
*   **Limitations:** Requires multiple rendering passes; bucket boundary determination is computationally intensive if not optimized via the two-pass technique.

## Background
The method was introduced by Fernandes and Walter [S2, unofficial]. It is cited in contrast to Zellmann et al.'s approach, which divides the volume into a uniform grid with a fixed brick size [S1, unofficial]. The technique is part of the broader field of volume rendering optimization, specifically targeting efficient empty-space skipping.

## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
[S2] Image-space sheet-buffered splatting on the GPU, https://openalex.org/W2419486389, fetched 2026-10-27 (unofficial)
