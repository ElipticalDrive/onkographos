# Hybrid Coarse-to-Fine Strategy

Also called: Multi-stage refinement, Progressive point cloud training

## Overview
The Hybrid Coarse-to-Fine Strategy is a training methodology for point-based radiance field representations that divides optimization into distinct coarse and fine stages, rather than training the point cloud in a single uniform pass [S1]. It addresses the instability and poor convergence of single-stage training by first establishing a global structural approximation and then progressively refining local details through densification [S1]. This approach yields significantly higher rendering quality compared to single-stage baselines, as evidenced by improved PSNR, SSIM, and LPIPS metrics on standard benchmarks [S1].

## How it works
The strategy operates by initializing a sparse representation and iteratively densifying it based on error metrics, while maintaining differentiability for end-to-end optimization [S1]. The specific implementation details for the densification phase are derived from the "Coarse-to-Fine Densification" variant described in [S2], which serves as the primary mechanism for this strategy in the context of primitive-based reconstruction.

### Inputs and Outputs
*   **Inputs:** A set of input images (potentially HDR), camera poses, and an initial sparse mesh or point cloud.
*   **Outputs:** A refined set of primitives (vertices/triangles) with optimized attributes (position, color, opacity, etc.) and a differentiable renderer.

### Data Structures
*   **Mesh/Point Cloud:** A collection of vertices $V$ and triangles $T$.
*   **Triangle Attributes:** Each triangle $t$ stores vertex features, pixel coverage count $|\text{pixels}_t|$, and accumulated error metrics.
*   **Error Metric:** Per-pixel Structural Dissimilarity (DSSIM) is computed for each triangle's projected area.

### Steps
1.  **Initialization (Coarse Phase):**
    *   Compute Sobel gradient magnitudes on a downsampled target image [S2].
    *   Sample initial vertex positions proportionally to edge strength, applying a small uniform floor to ensure coverage [S2].
    *   Create a Delaunay triangulation of the sampled points, adding boundary vertices to ensure full image coverage [S2].
    *   Initialize vertex features (e.g., color, opacity) [S2].

2.  **Iterative Refinement (Fine Phase):**
    *   Perform standard differentiable rendering and backpropagation to update vertex attributes [S1, S2].
    *   **Densification Trigger:** Every 500 iterations, starting at iteration 1,500 and ending at iteration 15,000, execute the densification step [S2].
    *   **Scoring:** For each triangle $t$, calculate a score using the formula:
        $$ \text{score}_t = \overline{\text{DSSIM}}_t \cdot |\text{pixels}_t|^\alpha $$
        where $\overline{\text{DSSIM}}_t$ is the mean DSSIM of triangle $t$, $|\text{pixels}_t|$ is the number of pixels covered by the triangle, and $\alpha = 0.75$ is the area-weighting exponent [S2].
    *   **Filtering:** Exclude triangles with fewer than 3 pixels from scoring [S2].
    *   **Vertex Insertion:** Identify the top-scoring triangles. Insert centroid vertices for these triangles. The number of new vertices is capped at $0.35 \times V$, where $V$ is the current vertex count [S2].
    *   **Feature Initialization:** Initialize the features of new centroid vertices by averaging the features of the parent triangle's vertices [S2].
    *   **Re-triangulation:** Re-run Delaunay triangulation on the full updated vertex set [S2].
    *   **Degeneracy Check:** Detect if the mesh has become degenerate (e.g., colinear vertices) and re-mesh if necessary, though this is rare [S2].

### Parameters
*   **Densification Interval:** 500 iterations [S2].
*   **Start Iteration:** 1,500 [S2].
*   **End Iteration:** 15,000 [S2].
*   **Area-weighting exponent ($\alpha$):** 0.75 [S2].
*   **Vertex Insertion Ratio:** 0.35 (new vertices per existing vertex) [S2].
*   **Minimum Pixel Coverage:** 3 pixels [S2].

### Edge Cases and Pitfalls
*   **Degeneracy:** Vertices may become colinear during densification, leading to degenerate triangles. The method includes a check to re-mesh in such cases [S2].
*   **Tiny Triangles:** Triangles with very few pixels can have high error but low spatial impact. The area-weighting exponent $\alpha < 1$ mitigates this by reducing the score of small triangles, preventing over-densification in small areas [S2].
*   **Precision:** The method assumes standard floating-point precision for gradient computation and DSSIM calculation. HDR images may require compression (e.g., to 14-bit white level) before supervision to emphasize shadow detail [S2].

## Subtopics

### Coarse-to-Fine Densification
A specific implementation of the hybrid strategy where a sparse mesh is progressively refined by inserting centroid vertices for high-scoring triangles and re-running Delaunay triangulation [S2]. It is chosen when a mesh-based representation is used instead of a pure point cloud. It sets itself apart by explicitly using Delaunay triangulation and area-weighted DSSIM scoring to guide densification [S2].

## Operating characteristics
*   **Time Complexity:** Training time is dominated by the differentiable rasterization and backpropagation steps. The densification step (Delaunay triangulation) adds overhead but is performed infrequently (every 500 iterations) [S2].
*   **Memory Complexity:** Memory usage scales with the number of vertices and triangles. The densification process increases the number of primitives, requiring more memory for attributes and gradients [S2].
*   **Performance:** On the Blender dataset, using the hybrid strategy achieves a PSNR of 30.3 dB, SSIM of 0.945, and LPIPS of 0.078 [S1]. Removing the strategy ("w/o hybrid") reduces PSNR to 26.6 dB, SSIM to 0.925, and increases LPIPS to 0.163 [S1].
*   **Hardware:** Training typically takes 1-3 minutes on an RTX A6000 GPU [S2].
*   **Strengths:** Improves rendering quality over single-stage training; prevents instability in early training; effectively distributes vertices based on error and area [S1, S2].
*   **Limitations:** Requires careful tuning of densification parameters (interval, ratio, alpha); Delaunay triangulation can be computationally expensive for very large meshes [S2].

## Background
The Hybrid Coarse-to-Fine Strategy originates from research into differentiable point-based radiance fields, which aim to provide an alternative to volumetric neural rendering by using points without sacrificing end-to-end differentiability [S1]. It was introduced to address the limitations of single-stage training, which often fails to capture both global structure and fine details simultaneously [S1]. The specific densification mechanics described in [S2] are part of the "Neural Harmonic Textures" method, which applies similar coarse-to-fine principles to primitive-based neural reconstruction.

## Sources
[S1] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
