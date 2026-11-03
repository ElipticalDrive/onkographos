# Mesh Reconstruction Evaluation


## Overview
Mesh Reconstruction Evaluation quantifies the geometric fidelity of a reconstructed triangular mesh against a ground-truth reference mesh [S1]. It solves the problem of assessing how well a surface reconstruction method captures the true shape, topology, and detail of the underlying scene [S1]. Unlike radiance-field metrics that evaluate visual appearance, this method focuses strictly on spatial accuracy, distinguishing it from [PSNR Metric Calculation](2903-psnr-metric-calculation.md) which measures pixel-level photometric error [S1].

## How it works
The evaluation computes statistical distances between the vertices of the reconstructed mesh and the ground-truth surface [S1]. The primary metrics are Chamfer Distance and F-Score [S1].

**Inputs:** Reconstructed mesh $\mathcal{M}_{rec}$ and ground-truth mesh $\mathcal{M}_{gt}$, typically represented as sets of vertices and faces [G].
**Outputs:** Scalar scores for Chamfer Distance and F-Score [G].
**Data Structures:** Point clouds or vertex sets for distance queries; KD-trees or spatial hashing for efficient nearest-neighbor search [G].
**Steps:**
1. Sample points uniformly from the surface of $\mathcal{M}_{rec}$ and $\mathcal{M}_{gt}$ to obtain point sets $P_{rec}$ and $P_{gt}$ [G].
2. Compute the Chamfer Distance (CD) as the sum of squared nearest-neighbor distances in both directions: $CD(P_{rec}, P_{gt}) = \frac{1}{|P_{rec}|} \sum_{p \in P_{rec}} \min_{q \in P_{gt}} \|p - q\|^2 + \frac{1}{|P_{gt}|} \sum_{q \in P_{gt}} \min_{p \in P_{rec}} \|p - q\|^2$ [G].
3. Compute the F-Score at a threshold distance $\tau$ by calculating precision (fraction of $P_{rec}$ within $\tau$ of $P_{gt}$) and recall (fraction of $P_{gt}$ within $\tau$ of $P_{rec}$), then taking the harmonic mean: $F(\tau) = \frac{2 \cdot \text{precision} \cdot \text{recall}}{\text{precision} + \text{recall}}$ [G].
**Parameters:** Threshold $\tau$ for F-Score, typically $0.01$ to $0.02$ of the bounding box diagonal [G]. Number of sampled points per mesh, typically $10^4$ to $10^5$ [G].
**Edge Cases and Pitfalls:**
- **Over-explanation:** The method focuses heavily on geometric details, which can cause over-explanation of texture on flat surfaces with complex geometry, leading to visible artifacts [S1].
- **Metric Discrepancy:** Quantitative mesh metrics (F-score, Chamfer distance) are good, but visualizations show apparent artifacts compared to ground truth [S1]. Evaluators should always inspect visual results alongside metrics [S1].

## Subtopics

### [Developability Regularizer](3292-developability-regularizer.md)
A mesh regularization term that encourages geometric flatness by minimizing the smallest eigenvalue of the vertex normal covariance matrix, weighted by vertex area. It distinguishes itself from smoothness-based regularizers like Laplacian smoothing by explicitly penalizing curvature to promote planar patches. Choose it when differentiable surface reconstruction requires explicit flatness constraints. It builds on [Mesh Reconstruction Evaluation](2909-mesh-reconstruction-evaluation.md).

## Operating characteristics

## Background

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
