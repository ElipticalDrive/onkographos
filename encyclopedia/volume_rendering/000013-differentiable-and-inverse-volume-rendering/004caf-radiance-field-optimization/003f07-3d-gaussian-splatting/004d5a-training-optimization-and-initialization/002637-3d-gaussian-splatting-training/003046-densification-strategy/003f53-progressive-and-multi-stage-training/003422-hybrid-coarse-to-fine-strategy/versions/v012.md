# Hybrid Coarse-to-Fine Strategy

Also called: Hybrid coarse-to-fine training, Multi-stage point cloud refinement

## Overview
A training strategy for point-based radiance field representations that divides optimization into distinct coarse and fine stages to improve rendering quality over single-stage training [S1]. It addresses the instability of optimizing sparse point clouds by first establishing a stable global structure before refining high-frequency details [S1]. Unlike single-stage approaches, it yields significantly higher PSNR and SSIM, and lower LPIPS [S1].

## How it works
**Inputs:** A set of input images with known camera poses; a target representation (point cloud or mesh) initialized sparsely.
**Outputs:** Optimized primitive attributes (position, color, opacity, etc.) and geometry.

**Data Structures:**
*   **Point Cloud/Mesh:** A collection of primitives. In the specific "Coarse-to-Fine Densification" variant, this is a Delaunay triangulation mesh where vertices hold features and triangles define connectivity [S2].
*   **Score Buffer:** Stores error metrics (e.g., DSSIM) per primitive [S2].

**Parameters:**
*   **Densification Interval:** 500 iterations [S2].
*   **Start Iteration:** 1,500 [S2].
*   **End Iteration:** 15,000 [S2].
*   **Area-weighting exponent ($\alpha$):** 0.75 [S2].
*   **Vertex Insertion Ratio:** 0.35 (max new vertices per step relative to current count $V$) [S2].
*   **Minimum Pixel Threshold:** 3 pixels per triangle for scoring [S2].

**Steps:**

1.  **Initialization (Coarse Stage):**
    *   Compute Sobel gradient magnitudes on a downsampled target image [S2].
    *   Sample initial vertex positions proportionally to edge strength, applying a small uniform floor to ensure coverage [S2].
    *   Create a Delaunay triangulation of sampled points, adding boundary vertices to ensure full image coverage [S2].
    *   Initialize primitive features (e.g., color) for these vertices [S2].

2.  **Iterative Refinement (Fine Stage):**
    *   For each iteration $i$ from 1 to the final iteration count:
        *   Render the current primitives to generate a predicted image.
        *   Compute the loss between the predicted image and the ground truth image.
        *   Update primitive attributes via backpropagation [S1, S2].
        *   **Densification Check:** If $i$ is a multiple of 500 and $1,500 \le i \le 15,000$:
            *   Calculate the score for each triangle $t$ using Equation 1 [S2]:
                $$ \text{score}_t = \overline{\text{DSSIM}}_t \cdot |\text{pixels}_t|^\alpha \quad (1) $$
                Where $\overline{\text{DSSIM}}_t$ is the mean per-pixel structural dissimilarity of triangle $t$, $|\text{pixels}_t|$ is the number of pixels covered by triangle $t$, and $\alpha = 0.75$ [S2].
            *   Exclude triangles with fewer than 3 pixels from scoring [S2].
            *   Identify the top-scoring triangles.
            *   Insert centroid vertices for these triangles. The number of new vertices is capped at $0.35 \times V$, where $V$ is the current vertex count [S2].
            *   Initialize features of new vertices by averaging the features of the parent triangle's vertices [S2].
            *   Re-run Delaunay triangulation on the updated vertex set [S2].
            *   Check for degeneracy (e.g., colinear vertices) and re-mesh if necessary [S2].

**Edge Cases and Pitfalls:**
*   **Degeneracy:** Triangles may become degenerate (colinear vertices) during densification. The method includes a check to re-mesh in such cases, though it is rare in practice [S2].
*   **Tiny Triangles:** The area-weighting exponent $\alpha$ prevents repeated splitting of tiny, high-error triangles, ensuring even distribution [S2].
*   **Precision:** The method relies on accurate gradient computation and DSSIM calculation. Using HDR compression (e.g., 14-bit white level) may be applied before supervision to emphasize shadow detail [S2].

## Subtopics

### Coarse-to-Fine Densification
A specific implementation of the hybrid strategy using mesh-based primitives. It initializes a sparse Delaunay mesh via edge-aware sampling and progressively refines it by inserting centroid vertices into high-error triangles every 500 iterations [S2]. It is chosen when a mesh representation is preferred over a point cloud, offering structured connectivity and explicit geometric refinement [S2].

## Operating characteristics
*   **Time Complexity:** Training time is significantly reduced compared to volumetric methods, taking 1-3 minutes on an RTX A6000 for typical scenes [S2]. The densification step adds overhead but is sparse (every 500 iterations) [S2].
*   **Memory Complexity:** Depends on the number of primitives. Densification increases memory usage linearly with the number of inserted vertices [S2].
*   **Performance:** On the Blender dataset, the hybrid strategy achieves 30.3 dB PSNR, 0.945 SSIM, and 0.078 LPIPS. Removing it ("w/o hybrid") drops PSNR to 26.6 dB, SSIM to 0.925, and increases LPIPS to 0.163 [S1].
*   **Strengths:** Stabilizes optimization, prevents poor local minima, and improves high-frequency detail recovery [S1, S2].
*   **Limitations:** Requires careful tuning of densification parameters (interval, ratio, thresholds) [S2].

## Background
Originated in differentiable point-based radiance field research as an alternative to neural volumetric representations [S1]. The hybrid coarse-to-fine approach was introduced to address the instability of single-stage training for sparse point clouds [S1]. The mesh-based variant (Coarse-to-Fine Densification) was later proposed for high-quality primitive-based neural reconstruction [S2].

## Sources
[S1] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
