# Mesh Reconstruction Evaluation

Also called: Mesh Quality Metrics, Surface Reconstruction Metrics

## Overview
Mesh Reconstruction Evaluation quantifies the geometric fidelity of a reconstructed triangular mesh against a ground-truth reference mesh. It solves the problem of assessing how well a surface reconstruction method captures the true shape, topology, and detail of the underlying scene. Unlike radiance-field metrics that evaluate visual appearance, this method focuses strictly on spatial accuracy, distinguishing it from [PSNR Metric Calculation](2903-psnr-metric-calculation.md) which measures pixel-level photometric error.

## How it works
The evaluation computes statistical distances between the vertices of the reconstructed mesh and the ground-truth surface. The primary metrics are Chamfer Distance and F-Score.

**Inputs:**
- Reconstructed mesh $M_{rec}$ (vertices $V_{rec}$, faces $F_{rec}$).
- Ground-truth mesh $M_{gt}$ (vertices $V_{gt}$, faces $F_{gt}$).

**Outputs:**
- Chamfer Distance ($CD$).
- F-Score ($F$).

**Data Structures:**
- Point clouds sampled from meshes: $P_{rec}$ and $P_{gt}$. Each point is a 3D coordinate $(x, y, z)$.
- Distance matrix: A matrix $D$ where $D_{ij}$ is the Euclidean distance between point $i$ in $P_{rec}$ and point $j$ in $P_{gt}$.

**Steps:**
1. **Sampling:** Uniformly sample points from the surface of $M_{rec}$ and $M_{gt}$ to create point sets $P_{rec}$ and $P_{gt}$. Let $N_{rec}$ and $N_{gt}$ be the number of samples. [G] Standard practice uses 10,000–100,000 points per mesh to balance accuracy and performance.
2. **Nearest Neighbor Search:** For each point $p \in P_{rec}$, find the nearest neighbor $q \in P_{gt}$ and compute the distance $d_{rec \to gt} = \|p - q\|$. Similarly, for each $q \in P_{gt}$, find the nearest neighbor $p \in P_{rec}$ and compute $d_{gt \to rec} = \|q - p\|$.
3. **Chamfer Distance Calculation:** Compute the mean squared error of these nearest-neighbor distances.
   $$ CD = \frac{1}{N_{rec}} \sum_{p \in P_{rec}} \min_{q \in P_{gt}} \|p - q\|^2 + \frac{1}{N_{gt}} \sum_{q \in P_{gt}} \min_{p \in P_{rec}} \|q - p\|^2 $$
   Lower values indicate better geometric alignment.
4. **Precision and Recall Calculation:** Define a threshold $\tau$ (e.g., 0.1 mm or 1% of scene bounding box diagonal).
   - Precision: The fraction of points in $P_{rec}$ whose nearest neighbor in $P_{gt}$ is within $\tau$.
     $$ Precision = \frac{|\{p \in P_{rec} : \min_{q \in P_{gt}} \|p - q\| < \tau\}|}{N_{rec}} $$
   - Recall: The fraction of points in $P_{gt}$ whose nearest neighbor in $P_{rec}$ is within $\tau$.
     $$ Recall = \frac{|\{q \in P_{gt} : \min_{p \in P_{rec}} \|q - p\| < \tau\}|}{N_{gt}} $$
5. **F-Score Calculation:** Compute the harmonic mean of Precision and Recall.
   $$ F = \frac{2 \cdot Precision \cdot Recall}{Precision + Recall} $$
   Higher values (max 1.0) indicate better overlap.

**Parameters:**
- $\tau$: Distance threshold for F-Score. Typical value: 0.1 mm for high-precision scans, or relative to scene scale. [G] If not specified, 1% of the bounding box diagonal is a common default.

**Edge Cases and Pitfalls:**
- **Over-explanation:** The method focuses heavily on geometric details, which can cause over-explanation of texture on flat surfaces with complex geometry, leading to visible artifacts [S1].
- **Metric Discrepancy:** Quantitative metrics like F-Score and Chamfer Distance may appear good, but visualizations can reveal apparent artifacts compared to ground truth [S1]. Evaluators should always inspect visual results alongside metrics.
- **Sampling Bias:** Insufficient sampling may miss small geometric features or holes.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N_{rec} \log N_{gt} + N_{gt} \log N_{rec})$ using efficient nearest-neighbor structures (e.g., KD-trees or Ball trees). With brute force, it is $O(N_{rec} N_{gt})$.
- **Memory Complexity:** $O(N_{rec} + N_{gt})$ to store point clouds.
- **Strengths:** Provides intuitive, interpretable measures of surface accuracy. F-Score is robust to outliers compared to Chamfer Distance.
- **Limitations:** Does not account for normal orientation or topological errors (e.g., holes, self-intersections) directly. Sensitive to sampling density.
- **When to use:** When geometric accuracy is the primary concern, such as in CAD reconstruction or architectural modeling.
- **When not to use:** When visual appearance (texture, lighting) is more important than geometry; use [PSNR Metric Calculation](2903-psnr-metric-calculation.md) instead.

## Background
Mesh reconstruction evaluation metrics like Chamfer Distance and F-Score are standard in computer vision and graphics for assessing 3D surface reconstruction quality. They are widely used in benchmarks for NeRF-based reconstruction, SLAM, and photogrammetry. The specific pitfalls of over-explanation and metric-visual discrepancy are noted in recent evaluations of radiance field rendering methods [S1].

## Sources
[S1] Sparse Voxels Rasterization: Real-time High-fidelity Radiance Field Rendering, https://doi.org/10.1109/cvpr52734.2025.01509, fetched 2026-10-26
