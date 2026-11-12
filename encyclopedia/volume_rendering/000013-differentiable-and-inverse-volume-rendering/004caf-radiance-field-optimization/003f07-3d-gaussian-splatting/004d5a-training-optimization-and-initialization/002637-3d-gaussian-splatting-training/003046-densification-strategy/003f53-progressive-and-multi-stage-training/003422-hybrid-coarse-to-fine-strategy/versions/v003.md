# Hybrid Coarse-to-Fine Strategy

Also called: Hybrid coarse-to-fine densification, multi-stage point cloud refinement

## Overview
A training strategy for point-based radiance fields that refines the primitive distribution through multiple stages of increasing density, rather than optimizing a single static point cloud [S1]. It addresses the problem of converging to local minima with insufficient geometric detail by starting with a coarse representation and progressively densifying it in regions of high reconstruction error [S1]. It yields significantly higher rendering quality than single-stage training [S1].

## How it works
The method operates in two distinct conceptual modes depending on the specific implementation context: general point cloud densification (as described in [S1]) and mesh-based coarse-to-fine refinement (as described in [S2]). The general principle involves initializing a sparse structure and iteratively adding primitives based on error metrics.

### Inputs and Outputs
- **Inputs:** A set of initial primitives (points or mesh vertices), camera poses, input images, and optimization hyperparameters (learning rate, densification interval, threshold).
- **Outputs:** A refined set of primitives with optimized attributes (position, opacity, color, covariance) and improved rendering quality.

### Data Structures
- **Point Cloud/Mesh:** A collection of primitives. In the mesh variant [S2], this is a set of vertices $V$ and a triangulation $T$.
- **Error Metric:** Per-primitive or per-region error scores (e.g., DSSIM, gradient magnitude).

### Steps
1. **Initialization:** Start with a sparse representation.
   - For point clouds [S1]: Initialize from SfM points or sparse sampling.
   - For meshes [S2]: Compute Sobel gradient magnitudes on a downsampled target image. Sample initial vertex positions proportionally to edge strength with a small uniform floor for coverage. Create a Delaunay triangulation of sampled points with boundary vertices ensuring full image coverage [S2].
2. **Training Loop:** Optimize primitive attributes using differentiable rendering.
3. **Densification Check:** Periodically evaluate the need for new primitives.
   - Interval: Every 500 iterations [S2].
   - Range: Starting at iteration 1,500 and ending at 15,000 [S2].
4. **Scoring:** Score primitives based on reconstruction error.
   - For meshes [S2]: Score each triangle $t$ using:
     $$ \text{score}_t = \overline{\text{DSSIM}}_t \cdot |\text{pixels}_t|^\alpha $$
     where $\overline{\text{DSSIM}}_t$ is the mean DSSIM (per-pixel structural dissimilarity) of triangle $t$, $|\text{pixels}_t|$ is the number of pixels covered by the triangle, and $\alpha = 0.75$ is the area-weighting exponent [S2].
   - Exclusion: Triangles with fewer than 3 pixels are excluded from scoring [S2].
5. **Insertion:** Add new primitives to high-scoring regions.
   - For meshes [S2]: Insert centroid vertices for the top-scoring triangles, up to $0.35 \times V$ new vertices (where $V$ is the current count). Initialize their features by averaging the parent triangle's vertex features. Re-run Delaunay triangulation on the full vertex set [S2].
   - For point clouds [S1]: Insert new points in regions with high gradient magnitude or reconstruction error [S1, inferred from context of "densification"].
6. **Pruning:** Remove redundant or low-opacity primitives to maintain efficiency [S1, inferred from parent principles].
7. **Degeneracy Handling:** Detect when the mesh becomes degenerate (e.g., vertices becoming colinear) and re-mesh, although this rarely occurs in practice [S2].

### Parameters
- $\alpha$: Area-weighting exponent, typically $0.75$ [S2].
- Densification interval: 500 iterations [S2].
- Max new vertices per step: $0.35 \times V$ [S2].
- Start/End iterations: 1,500 / 15,000 [S2].

### Edge Cases and Pitfalls
- **Degeneracy:** Meshes can become degenerate if vertices become colinear. A re-meshing step is required, though it is rare [S2].
- **Tiny Triangles:** Triangles with very few pixels can have high error but low spatial impact. The area-weighting exponent $\alpha < 1$ prevents repeated splitting of these tiny triangles [S2].
- **Precision:** Ensure that DSSIM calculations are stable for small pixel counts [S2].

## Subtopics

### Coarse-to-Fine Densification
A specific implementation of the hybrid strategy using a mesh representation. It initializes a sparse mesh via edge-aware sampling and progressively refines it by inserting centroid vertices for high-scoring triangles and re-running Delaunay triangulation [S2]. Choose this when working with mesh-based primitives or when explicit geometric connectivity is beneficial. It differs from point-based methods by maintaining explicit triangle topology [S2].

## Operating characteristics
- **Time Complexity:** Training generally takes between 1-3 minutes on an RTX A6000 for the mesh variant [S2]. The densification step adds overhead due to re-triangulation and scoring.
- **Memory Complexity:** Increases with the number of primitives. The mesh variant requires storing vertex and triangle data structures [S2].
- **Performance:** On the Blender dataset, removing the hybrid strategy reduces PSNR from 30.3 dB to 26.6 dB, SSIM from 0.945 to 0.925, and increases LPIPS from 0.078 to 0.163 [S1].
- **Strengths:** Better rendering quality than single-stage training; handles fine details by focusing densification on high-error regions [S1, S2].
- **Limitations:** Requires careful tuning of densification intervals and thresholds; mesh-based variants may suffer from degeneracy [S2].
- **When to use:** When high-fidelity view synthesis is required and computational resources allow for multi-stage optimization [S1].
- **When not to use:** When real-time training is required or when the scene is very simple and a single-stage optimization suffices.

## Background
The hybrid coarse-to-fine strategy was introduced in the context of differentiable point-based radiance fields [S1] and neural harmonic textures [S2]. It departs from volumetric representations and coordinate-network representations that require ray-marching [S1]. The strategy is part of a broader trend in radiance-field optimization to manage complexity through progressive refinement [S1, S2].

## Sources
[S1] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
