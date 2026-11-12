# Hybrid Coarse-to-Fine Strategy

Also called: Hybrid coarse-to-fine densification, Multi-stage point cloud refinement

## Overview
The Hybrid Coarse-to-Fine Strategy is a training methodology for point-based radiance field representations that divides optimization into distinct phases to manage primitive density and stability [S1]. Unlike single-stage training, which often suffers from instability or suboptimal convergence, this approach initializes a coarse representation and progressively refines it through densification steps [S1]. It sets itself apart from sibling methods like [Curriculum Learning with Slice and Voxel Supervision](12921-curriculum-learning-with-slice-and-voxel-supervision.md) by focusing on the structural evolution of the primitive population (points or meshes) rather than alternating supervision signals [S1, S2].

## How it works
The strategy operates by initializing a sparse geometric structure and iteratively increasing its density based on rendering error metrics, while maintaining differentiability for end-to-end optimization [S1, S2].

### Inputs and Outputs
*   **Inputs:** A set of input images (and optionally depth/normal priors), initial camera poses, and hyperparameters for densification frequency and thresholds.
*   **Outputs:** A refined set of primitives (points or mesh vertices) with optimized attributes (position, color, opacity, spherical harmonics coefficients) and a trained radiance field model.

### Data Structures
The specific data structure depends on the variant:
1.  **Point Cloud (Standard):** A list of $N$ points, each with position $\mathbf{p} \in \mathbb{R}^3$, opacity $\alpha$, color $\mathbf{c}$ (encoded via Spherical Harmonics or RGB), and covariance $\Sigma$ (for Gaussian splatting) or radius [S1].
2.  **Mesh (Coarse-to-Fine Densification Variant):** A set of vertices $V$ and a connectivity list defining triangles $T$. Each vertex stores features (color, etc.) [S2].

### Numbered Steps

#### 1. Initialization
Initialize the primitive population sparsely.
*   **For Point Clouds:** Randomly sample points or use a coarse grid [S1].
*   **For Meshes:** Compute Sobel gradient magnitudes on a downsampled target image. Sample initial vertex positions proportionally to edge strength, applying a small uniform floor to ensure coverage. Create a Delaunay triangulation of these points, adding boundary vertices to ensure full image coverage [S2].

#### 2. Coarse Training
Optimize the initial sparse representation using standard differentiable rendering losses (e.g., L1 color loss, SSIM) for a predefined number of iterations or until convergence [S1, S2].

#### 3. Iterative Densification
Repeat the following cycle until the target iteration count or density is reached:

1.  **Scoring:** Evaluate the error of each primitive.
    *   For mesh variants, score each triangle $t$ using:
        $$ \text{score}_t = \overline{\text{DSSIM}}_t \cdot |\text{pixels}_t|^\alpha $$
        where $\overline{\text{DSSIM}}_t$ is the mean per-pixel structural dissimilarity of triangle $t$, $|\text{pixels}_t|$ is the number of pixels covered by the triangle, and $\alpha = 0.75$ is an area-weighting exponent [S2].
    *   Exclude primitives covering fewer than 3 pixels from scoring [S2].
2.  **Selection:** Identify the top-scoring primitives that require refinement.
3.  **Insertion:**
    *   **Mesh Variant:** Insert centroid vertices for the top-scoring triangles. The number of new vertices is limited to $0.35 \times V$, where $V$ is the current vertex count. Initialize new vertex features by averaging the parent triangle's vertex features. Re-run Delaunay triangulation on the full vertex set [S2].
    *   **Point Cloud Variant:** Split or clone high-error points, initializing new attributes based on the parent (e.g., averaging colors, copying positions with small jitter) [S1, G].
4.  **Re-optimization:** Continue training the expanded primitive set with the rendering loss [S1, S2].

### Parameters
*   **Densification Frequency:** Every 500 iterations (typical for mesh variants) [S2].
*   **Start/End Iterations:** Densification starts at iteration 1,500 and ends at 15,000 (typical for mesh variants) [S2].
*   **Area-weighting exponent ($\alpha$):** 0.75 [S2].
*   **Max New Vertices/Points:** 35% of current count ($0.35 \times V$) [S2].
*   **Minimum Pixel Coverage:** 3 pixels [S2].

### Edge Cases and Pitfalls
*   **Degeneracy:** Meshes may become degenerate (e.g., colinear vertices) during densification. A remeshing step should be triggered if degeneracy is detected, though this is rare in practice [S2].
*   **Tiny High-Error Triangles:** Without area weighting ($\alpha$), tiny triangles with high error may be repeatedly split, leading to uneven distribution. The exponent $\alpha=0.75$ mitigates this [S2].
*   **Precision:** Ensure gradient computation remains stable during the insertion phase, as the topology changes [S1, S2].

## Subtopics

### Coarse-to-Fine Densification
A variant specifically using mesh-based initialization and refinement. It initializes a sparse mesh via edge-aware sampling and refines it by inserting centroid vertices for high-scoring triangles based on DSSIM and area weighting [S2]. Choose this when the scene benefits from explicit geometric surface representation during initialization, or when dealing with HDR data where shadow detail preservation is critical [S2].

## Operating characteristics
*   **Time Complexity:** Higher than single-stage training due to iterative densification and re-triangulation (for mesh variants) or re-initialization (for point variants). Mesh variants require $O(V \log V)$ for Delaunay triangulation at each step [S2].
*   **Memory Complexity:** Increases linearly with the number of primitives as densification proceeds.
*   **Performance:** Significantly improves rendering quality. On the Blender dataset, removing the hybrid strategy reduces PSNR from 30.3 dB to 26.6 dB, SSIM from 0.945 to 0.925, and increases LPIPS from 0.078 to 0.163 [S1].
*   **Strengths:** Prevents local minima associated with dense initialization; ensures stable convergence; distributes primitives evenly based on error [S1, S2].
*   **Limitations:** Requires careful tuning of densification schedules and thresholds; computationally more expensive per iteration due to topology updates [S1, S2].

## Background
The Hybrid Coarse-to-Fine Strategy originates from differentiable point-based radiance field research, which seeks to provide an alternative to volumetric neural rendering by using points without sacrificing end-to-end differentiability [S1]. It was introduced to address the instability and poor convergence of single-stage training for sparse point clouds [S1]. The mesh-based variant (Coarse-to-Fine Densification) was later developed to leverage explicit geometric priors for high-quality primitive-based neural reconstruction, particularly in HDR contexts [S2].

## Sources
[S1] Differentiable Point-Based Radiance Fields for Efficient View Synthesis, https://doi.org/10.1145/3550469.3555413, fetched 2026-10-26
[S2] Neural Harmonic Textures for High-Quality Primitive Based Neural Reconstruction, https://doi.org/10.1007/978-3-032-37032-7_18, fetched 2026-10-26
