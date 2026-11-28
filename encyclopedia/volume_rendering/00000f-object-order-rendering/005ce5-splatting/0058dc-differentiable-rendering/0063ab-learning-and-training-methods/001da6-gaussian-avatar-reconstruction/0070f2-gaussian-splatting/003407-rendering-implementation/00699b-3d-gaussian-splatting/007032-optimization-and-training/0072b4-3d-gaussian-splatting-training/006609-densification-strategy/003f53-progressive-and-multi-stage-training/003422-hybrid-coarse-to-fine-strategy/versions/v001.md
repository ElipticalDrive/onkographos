# Hybrid Coarse-to-Fine Strategy


## Overview
A training strategy involving multiple stages of refinement for the point cloud, as opposed to training the point cloud in a single stage [S1]. It yields better rendering quality than single-stage training by progressively refining the geometric representation [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of initial vertices (e.g., from edge-aware sampling), target images, and optimization hyperparameters.
*   **Output:** A refined mesh or point cloud with optimized vertex positions and features.

**Data Structures:**
*   **Mesh:** A Delaunay triangulation of vertices.
*   **Vertices:** Each vertex has position and feature attributes (e.g., color, spherical harmonics coefficients).
*   **Triangles:** Defined by three vertices. Each triangle tracks the number of pixels it covers and its mean DSSIM.

**Steps:**
1.  **Initialization:**
    *   Compute Sobel gradient magnitudes on a downsampled target image [S2].
    *   Sample initial vertex positions proportionally to edge strength, with a small uniform floor for coverage [S2].
    *   Create a Delaunay triangulation of the sampled points, ensuring boundary vertices cover the full image [S2].

2.  **Iterative Refinement (Densification):**
    *   Repeat every 500 iterations, starting at iteration 1,500 and ending at iteration 15,000 [S2]:
        1.  **Score Triangles:** For each triangle $t$, calculate the score:
            $$ \text{score}_t = \overline{\text{DSSIM}}_t \cdot |\text{pixels}_t|^\alpha $$
            where $\overline{\text{DSSIM}}_t$ is the mean DSSIM (per-pixel structural dissimilarity) of triangle $t$, $|\text{pixels}_t|$ is the number of pixels covered by the triangle, and $\alpha = 0.75$ is the area-weighting exponent [S2].
        2.  **Filter:** Exclude triangles with fewer than 3 pixels from scoring [S2].
        3.  **Insert Vertices:** Identify the top-scoring triangles. Insert centroid vertices for these triangles, up to $0.35 \times V$ new vertices, where $V$ is the current vertex count [S2].
        4.  **Initialize Features:** Initialize the features of new vertices by averaging the features of the parent triangle's vertices [S2].
        5.  **Re-triangulate:** Re-run Delaunay triangulation on the full set of vertices [S2].
        6.  **Degeneracy Check:** Detect if the mesh becomes degenerate (e.g., colinear vertices) and re-mesh if necessary [S2].

**Parameters:**
*   $\alpha = 0.75$ (area-weighting exponent) [S2].
*   Densification interval: Every 500 iterations [S2].
*   Start iteration: 1,500 [S2].
*   End iteration: 15,000 [S2].
*   Max new vertices per step: $0.35 \times V$ [S2].

**Edge Cases and Pitfalls:**
*   **Degeneracy:** Vertices may become colinear, requiring re-meshing [S2].
*   **Small Triangles:** Triangles with fewer than 3 pixels are excluded to prevent noise-driven densification [S2].
*   **Precision:** The area weighting prevents repeated splitting of tiny, high-error triangles [S2].

## Subtopics

### Coarse-to-Fine Densification
A specific implementation of the hybrid coarse-to-fine strategy where a sparse mesh is progressively refined by inserting centroid vertices for high-scoring triangles and re-running Delaunay triangulation [S2]. It is chosen when a mesh-based representation is used instead of a point cloud. It differs from standard point cloud densification by operating on triangle scores and maintaining a Delaunay structure [S2].

## Operating characteristics
*   **Performance:** On the Blender dataset, removing the hybrid strategy ('w/o hybrid') reduces PSNR from 30.3 dB to 26.6 dB, SSIM from 0.945 to 0.925, and increases LPIPS from 0.078 to 0.163 [S1].
*   **Complexity:** Not given in the sources.
*   **Strengths:** Better rendering quality than single-stage training [S1].
*   **Limitations:** Requires multiple stages of refinement, potentially increasing training time compared to single-stage methods (though specific time savings are not quantified relative to single-stage, total training time is noted as 1-3 minutes on an RTX A6000 for the full method including CUDA rasterizer) [S1, S2].

## Background
Originated in the context of differentiable point-based radiance fields for efficient view synthesis [S1]. It is part of a broader trend towards non-volumetric, point-based or mesh-based radiance field representations that avoid the computational cost of ray-marching neural networks [S1]. The specific coarse-to-fine densification algorithm is detailed in the context of Neural Harmonic Textures for high-quality primitive-based neural reconstruction [S2].

## Sources
[S1] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
