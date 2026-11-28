# Hybrid Coarse-to-Fine Strategy

Also called: Hybrid coarse-to-fine densification, multi-stage point cloud refinement

## Overview
A training strategy that improves radiance field reconstruction quality by refining the primitive representation (point cloud or mesh) in multiple stages, rather than optimizing a single static configuration [S1, S2]. It addresses the problem of insufficient geometric detail in single-stage training, where primitives either fail to capture high-frequency details or become redundant in smooth regions [S1]. It sets itself apart from single-stage baselines by dynamically adjusting the density and distribution of primitives based on error metrics during training [S1].

## How it works
The method operates on a set of primitives (points or mesh vertices) $\mathcal{P}$. The core mechanism involves periodic densification steps triggered at specific iteration intervals, where primitives are added or modified based on a scoring function derived from rendering error.

**Inputs:**
*   Current primitive set $\mathcal{P}$ (points or mesh vertices).
*   Current iteration $I$.
*   Rendering loss $L$ (or per-pixel error metrics like DSSIM).
*   Configuration parameters: refinement interval, scoring exponent, max vertex count ratio.

**Outputs:**
*   Updated primitive set $\mathcal{P}'$.

**Data Structures:**
*   **Point Cloud (S1):** List of Gaussians. No explicit mesh topology. Densification involves splitting/cloning existing Gaussians.
*   **Mesh (S2):** List of vertices $V$ and triangles $T$. Topology is maintained via Delaunay triangulation.

**Numbered Steps:**

1.  **Initialization:**
    *   *Point Cloud (S1):* Initialize a sparse set of 3D Gaussians.
    *   *Mesh (S2):* Compute Sobel gradient magnitudes on a downsampled target image. Sample initial vertex positions proportionally to edge strength with a small uniform floor for coverage. Create a Delaunay triangulation of sampled points with boundary vertices ensuring full image coverage [S2].

2.  **Iteration Loop:**
    For each training iteration $I$:
    a. Render the current primitive set to obtain images.
    b. Compute loss and gradients.
    c. Update primitive parameters (positions, opacities, colors, etc.) via optimizer.

3.  **Densification Check:**
    If $I$ satisfies the refinement schedule:
    *   *Point Cloud (S1):* Trigger densification based on gradient magnitude and opacity thresholds (see [Parent](9966-progressive-and-multi-stage-training.md)).
    *   *Mesh (S2):* Proceed to Step 4.

4.  **Scoring (Mesh Variant S2):**
    For each triangle $t \in T$:
    a. Calculate mean DSSIM ($\overline{\text{DSSIM}}_t$) for pixels covered by $t$.
    b. Count pixels $|\text{pixels}_t|$ covered by $t$.
    c. Compute score:
       $$ \text{score}_t = \overline{\text{DSSIM}}_t \cdot |\text{pixels}_t|^\alpha $$
       where $\alpha = 0.75$ [S2].
    d. Exclude triangles with fewer than 3 pixels [S2].

5.  **Insertion/Refinement:**
    *   *Mesh (S2):* Identify top-scoring triangles. Insert centroid vertices for these triangles. Limit new vertices to $0.35 \times V$ (where $V$ is current vertex count) [S2]. Initialize new vertex features by averaging parent triangle vertex features. Re-run Delaunay triangulation on the full vertex set [S2]. Check for degeneracy (e.g., colinear vertices) and re-mesh if necessary [S2].
    *   *Point Cloud (S1):* Split or clone Gaussians based on gradient magnitude $\tau_k$ and opacity $\alpha_k$ thresholds [Parent].

**Parameters:**
*   **Refinement Interval:** Every 500 iterations for Mesh (S2); periodic for Point Cloud (S1, see Parent).
*   **Start/End Iterations (Mesh):** Start at 1,500, end at 15,000 (S2).
*   **Area-weighting exponent ($\alpha$):** 0.75 (S2).
*   **Vertex Growth Ratio:** 0.35 (S2).

**Edge Cases:**
*   **Degeneracy:** Mesh triangles may become degenerate (colinear vertices). Detected and re-meshed [S2].
*   **Small Triangles:** Triangles with <3 pixels are excluded from scoring to prevent noise-driven splitting [S2].

## Subtopics

### Coarse-to-Fine Densification
A specific implementation using a sparse mesh that is progressively refined by inserting centroid vertices for high-scoring triangles and re-running Delaunay triangulation [S2]. It differs from point-based splitting by maintaining explicit mesh topology and using DSSIM-based scoring with area weighting [S2]. Choose this when explicit geometric continuity or mesh-based rendering is required. [S2](https://doi.org/10.1007/978-3-032-37032-7_18)

## Operating characteristics
*   **Performance:** Significantly improves rendering quality over single-stage training. On the Blender dataset, removing the hybrid strategy reduces PSNR from 30.3 dB to 26.6 dB, SSIM from 0.945 to 0.925, and increases LPIPS from 0.078 to 0.163 [S1].
*   **Complexity:** Adds overhead for periodic densification and re-meshing (S2) or buffer compaction (S1). Mesh re-triangulation is $O(V \log V)$ or similar depending on implementation [S2].
*   **Strengths:** Captures fine details that single-stage training misses; prevents over-smoothing [S1, S2].
*   **Limitations:** Requires careful tuning of refinement schedules and thresholds; increased training time due to densification steps [S1, S2].

## Background
Originated in differentiable point-based radiance field methods to address the limitations of static point cloud initialization [S1]. The mesh-based variant was introduced in Neural Harmonic Textures to leverage explicit geometric structures for high-quality reconstruction [S2]. It builds on the general principle of progressive refinement in inverse volume rendering [Parent].

## Sources
[S1] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
