# Hybrid Coarse-to-Fine Strategy

Also called: Hybrid coarse-to-fine training, Multi-stage point cloud refinement

## Overview
A training strategy for point-based radiance field representations that partitions optimization into sequential stages of refinement, rather than training the point cloud in a single static phase. It solves the problem of local minima and poor geometric initialization by progressively densifying the representation based on error metrics. It yields significantly better rendering quality than single-stage training [S1].

## How it works
**Inputs:**
- A set of input images (typically downsampled for initialization).
- Initial sparse point cloud or mesh vertices (often derived from Structure-from-Motion or edge-aware sampling).
- Hyperparameters: Densification interval, start/end iterations, area-weighting exponent $\alpha$, max vertex insertion ratio.

**Outputs:**
- A refined set of points (or mesh vertices) with optimized attributes (position, color, opacity, etc.).

**Data Structures:**
- **Point Cloud/Mesh:** A collection of vertices $V$. In the mesh variant [S2], vertices are organized into triangles $T$ via Delaunay triangulation.
- **Error Buffer:** Per-triangle or per-point structural dissimilarity (DSSIM) values.

**Steps:**

1. **Initialization:**
   - Generate an initial sparse representation.
   - *Mesh Variant [S2]:* Compute Sobel gradient magnitudes on a downsampled target image. Sample initial vertex positions proportionally to edge strength with a small uniform floor for coverage. Create a Delaunay triangulation of sampled points, adding boundary vertices to ensure full image coverage.

2. **Iterative Refinement Loop:**
   - For each training iteration $i$:
     - Render the current representation to generate predicted images.
     - Compute photometric and structural losses (e.g., DSSIM) against ground truth images.
     - Update representation attributes via gradient descent.

3. **Densification Trigger:**
   - Check if current iteration $i$ meets the densification schedule.
   - *Mesh Variant [S2]:* Trigger every 500 iterations, starting at iteration 1,500 and ending at 15,000.

4. **Scoring (Mesh Variant [S2]):**
   - For each triangle $t$ in the current mesh:
     - Calculate mean DSSIM: $\overline{\text{DSSIM}}_t$.
     - Count pixels covered by triangle: $|\text{pixels}_t|$.
     - Exclude triangles with fewer than 3 pixels.
     - Compute score:
       $$ \text{score}_t = \overline{\text{DSSIM}}_t \cdot |\text{pixels}_t|^\alpha $$
       where $\alpha = 0.75$ is the area-weighting exponent. This weighting prevents repeated splitting of tiny, high-error triangles and distributes vertices proportionally to spatial impact.

5. **Vertex Insertion (Mesh Variant [S2]):**
   - Identify top-scoring triangles.
   - Insert centroid vertices for these triangles.
   - Limit new vertices to $0.35 \times V$, where $V$ is the current vertex count.
   - Initialize new vertex features by averaging the parent triangle's vertex features.
   - Re-run Delaunay triangulation on the full vertex set.
   - *Degeneracy Check:* Detect if the mesh becomes degenerate (e.g., colinear vertices) and re-mesh if necessary [G].

**Parameters:**
- $\alpha = 0.75$ (Area-weighting exponent) [S2].
- Densification interval: 500 iterations [S2].
- Start iteration: 1,500 [S2].
- End iteration: 15,000 [S2].
- Max insertion ratio: 0.35 [S2].

**Edge Cases and Pitfalls:**
- **Degeneracy:** Triangles may become degenerate during densification. The mesh variant includes detection and re-meshing logic, though this rarely occurs in practice [S2].
- **Tiny Triangles:** Triangles with fewer than 3 pixels are excluded from scoring to prevent noise-driven densification [S2].
- **Precision:** DSSIM computation should be stable; standard implementations use sliding window correlations [G].

## Subtopics

### Coarse-to-Fine Densification
A specific implementation of the hybrid strategy using a sparse mesh that is progressively refined. It initializes via edge-aware sampling (Sobel gradients) and densifies by inserting centroids of high-DSSIM triangles, re-running Delaunay triangulation. It is chosen when a mesh-based representation is preferred over a raw point cloud for structured geometry. It differs from standard point densification by explicitly managing topological connectivity via triangulation [S2].

## Operating characteristics
- **Performance:** On the Blender dataset, removing the hybrid strategy reduces PSNR from 30.3 dB to 26.6 dB, SSIM from 0.945 to 0.925, and increases LPIPS from 0.078 to 0.163 [S1].
- **Time Complexity:** Training generally takes 1-3 minutes on an RTX A6000 for the mesh variant [S2]. The repeated Delaunay triangulation adds overhead compared to simple point insertion [S2].
- **Memory Complexity:** Increases as the number of vertices/triangles grows during densification. The limit of $0.35 \times V$ per step controls memory growth [S2].
- **Strengths:** Prevents local minima; achieves higher fidelity than single-stage training; adapts density to scene complexity [S1, S2].
- **Limitations:** Requires careful scheduling of densification steps; mesh variant has higher computational cost per step due to triangulation [S2].

## Background
Originated in differentiable point-based radiance field research as an alternative to volumetric neural rendering [S1]. The mesh-based coarse-to-fine densification was introduced in the context of Neural Harmonic Textures for high-quality primitive-based reconstruction [S2]. It builds on the principle that sparse initializations benefit from error-guided refinement rather than uniform densification.

## Sources
[S1] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
