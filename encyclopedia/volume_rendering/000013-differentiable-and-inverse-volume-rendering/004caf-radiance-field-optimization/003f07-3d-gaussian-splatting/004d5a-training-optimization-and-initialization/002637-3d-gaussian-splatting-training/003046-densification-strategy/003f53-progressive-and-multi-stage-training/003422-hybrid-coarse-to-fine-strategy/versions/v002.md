# Hybrid Coarse-to-Fine Strategy

Also called: Coarse-to-Fine Densification, Multi-stage Refinement

## Overview
The Hybrid Coarse-to-Fine Strategy is a training methodology for point-based radiance field representations that employs multiple stages of refinement rather than a single-stage optimization [S1]. It addresses the problem of insufficient geometric detail and poor reconstruction quality in sparse initial point clouds by progressively densifying the representation based on error metrics [S1]. Unlike single-stage training, this approach yields significantly higher rendering quality by balancing global structure capture with local detail refinement [S1].

## How it works
The strategy operates within a differentiable training loop, utilizing a mesh-based or point-based representation that is iteratively refined. The core mechanism involves scoring primitives based on reconstruction error and spatial extent, then inserting new primitives to reduce error in high-impact regions [S2].

**Inputs and Outputs:**
*   **Input:** Current set of primitives (triangles or points), target image, current iteration count.
*   **Output:** Updated set of primitives with higher density in error-prone areas.

**Data Structures:**
*   **Mesh/Point Set:** A collection of vertices and connectivity information (e.g., Delaunay triangulation) [S2].
*   **Buffers:** GPU buffers storing vertex positions, features, and connectivity [S2].

**Steps:**

1.  **Initialization:**
    *   Compute Sobel gradient magnitudes on a downsampled target image [S2].
    *   Sample initial vertex positions proportionally to edge strength, applying a small uniform floor to ensure coverage [S2].
    *   Create a Delaunay triangulation of the sampled points, adding boundary vertices to ensure full image coverage [S2].

2.  **Periodic Densification Trigger:**
    *   Densification is triggered every 500 iterations [S2].
    *   The process starts at iteration 1,500 and ends at iteration 15,000 [S2].

3.  **Scoring:**
    *   For each triangle $t$, compute a score using the formula:
        $$ \text{score}_t = \overline{\text{DSSIM}}_t \cdot |\text{pixels}_t|^\alpha $$
    *   Where:
        *   $\overline{\text{DSSIM}}_t$ is the mean DSSIM (per-pixel structural dissimilarity) of triangle $t$ [S2].
        *   $|\text{pixels}_t|$ is the number of pixels covered by triangle $t$ [S2].
        *   $\alpha = 0.75$ is the area-weighting exponent [S2].
    *   Exclude triangles with fewer than 3 pixels from scoring [S2].

4.  **Vertex Insertion:**
    *   Identify the top-scoring triangles [S2].
    *   Insert centroid vertices for these triangles [S2].
    *   Limit the number of new vertices to $0.35 \times V$, where $V$ is the current vertex count [S2].
    *   Initialize features of new vertices by averaging the features of the parent triangle's vertices [S2].

5.  **Re-triangulation:**
    *   Re-run Delaunay triangulation on the full set of vertices (old and new) [S2].
    *   Detect and handle degenerate meshes (e.g., colinear vertices) by re-meshing if necessary [S2].

**Parameters:**
*   $\alpha = 0.75$ [S2].
*   Densification interval: 500 iterations [S2].
*   Start iteration: 1,500 [S2].
*   End iteration: 15,000 [S2].
*   Vertex insertion limit: $0.35 \times V$ [S2].

**Edge Cases and Pitfalls:**
*   **Degeneracy:** Vertices may become colinear, requiring re-meshing [S2].
*   **Tiny Triangles:** Triangles with fewer than 3 pixels are excluded to prevent over-splitting [S2].
*   **Precision:** DSSIM computation requires careful handling of numerical stability [S2].

## Subtopics

### Coarse-to-Fine Densification
A specific implementation of the hybrid strategy using a sparse mesh that is progressively refined by inserting centroid vertices for high-scoring triangles and re-running Delaunay triangulation [S2]. It is chosen when a mesh-based representation is preferred for its structured connectivity and efficient rasterization [S2]. It differs from point-based methods by maintaining explicit triangle connectivity [S2].

## Operating characteristics
*   **Time Complexity:** Densification occurs periodically (every 500 iterations), adding overhead to the training loop [S2]. Re-triangulation is a non-linear operation dependent on the number of vertices [S2].
*   **Memory Complexity:** Memory usage increases as vertices are added, limited by the $0.35 \times V$ growth factor per step [S2].
*   **Performance:** On the Blender dataset, removing the hybrid strategy reduces PSNR from 30.3 dB to 26.6 dB, SSIM from 0.945 to 0.925, and increases LPIPS from 0.078 to 0.163 [S1].
*   **Strengths:** Improves rendering quality significantly compared to single-stage training [S1]. Prevents over-smoothing by focusing refinement on high-error areas [S2].
*   **Limitations:** Requires careful tuning of hyperparameters (e.g., $\alpha$, iteration bounds) [S2]. Computationally more expensive than single-stage training due to periodic densification and re-triangulation [S2].

## Background
The Hybrid Coarse-to-Fine Strategy originates from research into differentiable point-based radiance fields for efficient view synthesis [S1]. It was introduced to address the limitations of single-stage training, which often fails to capture fine geometric details [S1]. The mesh-based variant, Coarse-to-Fine Densification, was further developed in the context of neural harmonic textures for high-quality primitive-based neural reconstruction [S2].

## Sources
[S1] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
