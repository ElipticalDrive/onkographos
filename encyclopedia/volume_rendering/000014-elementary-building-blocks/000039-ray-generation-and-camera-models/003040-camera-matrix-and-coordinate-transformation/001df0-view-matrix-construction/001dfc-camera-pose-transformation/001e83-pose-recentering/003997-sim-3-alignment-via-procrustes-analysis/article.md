# Sim(3) Alignment via Procrustes Analysis

Also called: Camera Pose Alignment, Pose Accuracy Measurement, Procrustes alignment, Sim(3) BARF

## Overview
Sim(3) Alignment via Procrustes Analysis computes a 3D similarity transformation (rotation, uniform scaling, and translation) to align estimated camera poses to ground truth poses by minimizing the sum of squared Euclidean distances between corresponding camera center locations [S2, unofficial]. It serves as the primary method for pose quality assessment and initialization in neural rendering pipelines, distinguishing itself from rigid-only alignment by accounting for scale discrepancies and from iterative methods by providing a closed-form solution [S2, unofficial].

## How it works
**Inputs:**
- Estimated camera poses $\{[R_i | t_i]\}_{i=1}^N$ and ground truth poses $\{[R_i^{gt} | t_i^{gt}]\}_{i=1}^N$ [S1, unofficial].
- Correspondence is established by index $i$ [G].

**Outputs:**
- A similarity transformation $T \in \text{Sim}(3)$ defined by a rotation matrix $R \in \mathbb{R}^{3 \times 3}$, a scale factor $s \in \mathbb{R}$, and a translation vector $t \in \mathbb{R}^3$ [S1, unofficial].
- The aligned estimated poses $C'_{est}$, where $c'_i = s R c_i + t$ [S1, unofficial].

**Data Structures:**
- Two sets of 3D points: estimated centers $C_{est} = \{c_1, \dots, c_N\}$ and ground truth centers $C_{gt} = \{c_1^{gt}, \dots, c_N^{gt}\}$ [S1, unofficial].

**Steps:**
1. **Extract Camera Centers:** Compute world coordinates for each pose: $c_i = -R_i^\top t_i$ and $c_i^{gt} = -(R_i^{gt})^\top t_i^{gt}$ [S1, unofficial].
2. **Centroid Calculation:** Compute centroids of both sets: $\mu_{est} = \frac{1}{N} \sum_{i=1}^N c_i$ and $\mu_{gt} = \frac{1}{N} \sum_{i=1}^N c_i^{gt}$ [G].
3. **Centering:** Subtract centroids to obtain centered coordinates: $\tilde{c}_i = c_i - \mu_{est}$ and $\tilde{c}_i^{gt} = c_i^{gt} - \mu_{gt}$ [G].
4. **Covariance Matrix:** Compute the $3 \times 3$ matrix $H = \sum_{i=1}^N \tilde{c}_i (\tilde{c}_i^{gt})^\top$ [G].
5. **Singular Value Decomposition (SVD):** Compute $H = U \Sigma V^\top$ [G].
6. **Rotation Calculation:** Compute $R = V U^\top$ [G].
7. **Scale Calculation:** Compute $s = \frac{\text{trace}(\Sigma)}{\sum_{i=1}^N \|\tilde{c}_i\|^2}$ [G].
8. **Translation Calculation:** Compute $t = \mu_{gt} - s R \mu_{est}$ [G].
9. **Application:** Apply the transformation to align the poses: $c'_i = s R c_i + t$ [S1, unofficial].

**Parameters:**
- Not given in the sources [G].

**Edge Cases and Pitfalls:**
- **Degenerate Geometry:** Procrustes analysis is unreliable when all camera centers lie on a single line or if the translation vectors contain significant noise [S2, unofficial].
- **Precision:** Numerical stability depends on the conditioning of the covariance matrix $H$ [G].

## Subtopics

### Sim(3) with Rotation Alignment
Computes the optimal alignment by incorporating camera rotation via a virtual point to handle degenerate trajectories [S2, unofficial]. It sets itself apart from standard Sim(3) by reducing rotation errors at the cost of increased translation errors [S2, unofficial]. Choose it when camera centers are collinear or translation noise is high [S2, unofficial].

### [Test-time Optimization with Sim(3) Initialization](8565-test-time-optimization-with-sim-3-initialization.md)
A testing procedure involving Sim(3) alignment initialization followed by additional test-time optimization to adjust test poses for view synthesis. It sets itself apart from local neighbor proximity or unoptimized estimated poses by using a global rigid-similarity transformation derived from training data.

### [Pose Registration Metrics](12703-pose-registration-metrics.md)
Evaluates camera pose estimation accuracy using absolute translation and rotation errors. It sets itself apart by providing scalar values to assess methods like Sim(3) Alignment via Procrustes Analysis.

## Operating characteristics
- **Time and Memory Complexity:** Not given in the sources [G].
- **Measured Performance:** Not given in the sources [G].
- **Scaling:** Not given in the sources [G].
- **Strengths:** Accounts for scale discrepancies between estimated and ground truth trajectories; widely used for pose quality assessment in NeRF pipelines [S2, unofficial].
- **Limitations:** Fails or becomes unreliable if camera centers are collinear or if translation noise is high [S2, unofficial]. Standard Sim(3) causes fewer translation errors than Sim(3) with rotation when pose accuracy is high, whereas Sim(3) with rotation reduces rotation errors but increases translation errors [S2, unofficial].
- **When to use:** For evaluating pose accuracy or initializing test-time optimization in neural rendering [S2, unofficial].
- **When not to use:** When camera trajectories are degenerate (collinear) or when high-fidelity rotation alignment is required despite translation error penalties (consider Sim(3) with rotation) [S2, unofficial].

## Background
The method applies Procrustes analysis to camera pose alignment in neural radiance fields [S2, unofficial]. It is notably employed in BARF and L2G-NeRF for pose quality assessment alignment [S2, unofficial]. The approach was later critiqued and extended by CF-NeRF, which introduced the "Sim(3) with rotation" variant to address reliability issues in specific degenerate cases [S2, unofficial].

## Sources
[S1] SVRecon: Sparse Voxel Rasterization for Surface Reconstruction, https://doi.org/10.48550/arxiv.2511.17364, fetched 2026-10-26 (unofficial)
[S2] CT-NeRF: Incremental Optimizing Neural Radiance Field and Poses with Complex Trajectory, https://doi.org/10.48550/arxiv.2404.13896, fetched 2026-10-26 (unofficial)
[S3] DBARF: Deep Bundle-Adjusting Generalizable Neural Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00011, fetched 2026-10-29
