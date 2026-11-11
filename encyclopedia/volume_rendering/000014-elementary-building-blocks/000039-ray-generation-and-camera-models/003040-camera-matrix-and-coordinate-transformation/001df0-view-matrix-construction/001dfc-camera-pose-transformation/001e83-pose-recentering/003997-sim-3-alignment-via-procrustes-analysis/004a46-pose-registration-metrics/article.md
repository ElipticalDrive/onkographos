# Pose Registration Metrics

Also called: Pose Error Metrics, Rotation Error, Translation Error

## Overview
Pose Registration Metrics quantify the discrepancy between estimated camera poses and ground-truth poses after alignment, serving as the primary evaluation method for the global consistency of ego-motion estimation and SLAM systems [S1]. Unlike local metrics that measure relative drift between adjacent frames, these metrics assess the integrated distance of the entire trajectory, making them distinct from siblings like [Rigid Parametric Alignment](8609-rigid-parametric-alignment.md) which focuses on parametric fitting rather than error reporting [S1].

## How it works
The method computes two scalar error values: rotation error and translation error, averaged over a set of poses [S1].

**Inputs and Outputs:**
- Input: A set of $N$ ground-truth rotation matrices $R_i \in \mathbb{R}^{3 \times 3}$ and position vectors $T_i \in \mathbb{R}^3$.
- Input: A set of $N$ estimated rotation matrices $\hat{R}_i \in \mathbb{R}^{3 \times 3}$ and position vectors $\hat{T}_i \in \mathbb{R}^3$.
- Input: A Sim(3) alignment transformation $(s, R_{align}, T_{align})$ derived from [Sim(3) Alignment via Procrustes Analysis](8526-sim-3-alignment-via-procrustes-analysis.md) to align the estimated poses to the ground truth coordinate system [S1].
- Output: Average rotation error $|R_{err}|$ and average translation error $T_{err}$.

**Data Structures:**
- Rotation matrices are $3 \times 3$ orthogonal matrices with determinant +1.
- Position vectors are 3D Euclidean coordinates. Note that positions are often not in metric space (e.g., normalized coordinates), meaning the translation error has no physical units [S1].

**Steps:**
1. **Align Poses:** For each estimated pose $i$, apply the Sim(3) alignment transformation to the estimated position $\hat{T}_i$ and rotation $\hat{R}_i$.
   - Aligned position: $T'_i = s \cdot R_{align} \hat{T}_i + T_{align}$
   - Aligned rotation: $R'_i = R_{align} \hat{R}_i$
   [G] If the alignment provides a full 4x4 matrix, extract the upper-left 3x3 rotation and the translation column.

2. **Compute Per-Pose Rotation Error:** For each aligned pose $i$, calculate the rotation error $|R_{err, i}|$ using the trace of the relative rotation matrix:
   $$ |R_{err, i}| = \left| \cos^{-1} \left( \frac{\text{Tr}(R_i^{-1} R'_i) - 1}{2} \right) \right| $$
   where $\text{Tr}(\cdot)$ denotes the matrix trace [S1].
   [G] To ensure numerical stability, clamp the argument of $\cos^{-1}$ to the range $[-1, 1]$. The result is in radians.

3. **Compute Per-Pose Translation Error:** For each aligned pose $i$, calculate the translation error $T_{err, i}$ as the Euclidean distance between the ground-truth position $T_i$ and the aligned estimated position $T'_i$:
   $$ T_{err, i} = \| T'_i - T_i \|_2 $$
   [S1]

4. **Average Errors:** Compute the mean rotation error and mean translation error over all $N$ poses:
   $$ \text{Average } |R_{err}| = \frac{1}{N} \sum_{i=1}^{N} |R_{err, i}| $$
   $$ \text{Average } T_{err} = \frac{1}{N} \sum_{i=1}^{N} T_{err, i} $$
   [S1]

**Parameters:**
- No free parameters; the method is deterministic given the alignment.

**Edge Cases and Pitfalls:**
- **Non-Metric Space:** As noted in [S1], positions may not be in metric space. The translation error value is unitless and should not be interpreted as physical distance (e.g., meters) unless the data is explicitly scaled.
- **Rotation Ambiguity:** The formula uses the absolute value of the angle. The rotation error is bounded in $[0, \pi]$.
- **Numerical Precision:** The argument to $\cos^{-1}$ must be clamped to $[-1, 1]$ to avoid NaNs due to floating-point errors when $R_i$ and $R'_i$ are nearly identical.

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of poses. Each pose requires a constant number of matrix operations and a square root.
- **Memory Complexity:** $O(1)$ auxiliary space if computed incrementally, or $O(N)$ if storing per-pose errors.
- **Strengths:** Provides a single scalar summary of global trajectory consistency. Simple to compute and interpret.
- **Limitations:** Does not capture local inconsistencies or temporal drift patterns. Sensitive to the quality of the initial Sim(3) alignment; poor alignment inflates errors [S1].
- **When to use:** Evaluating the global accuracy of SLAM or pose estimation systems after alignment.
- **When not to use:** When local drift or temporal consistency is the primary concern; use local metrics instead.

## Background
The metrics originate from the evaluation framework for sparse and noisy pose estimation in neural radiance fields, specifically introduced in the SPARF paper [S1]. They are widely adopted in novel-view synthesis and SLAM literature to report average rotation and translation errors after aligning estimated trajectories to ground truth [S1].

## Sources
[S1] SPARF: Neural Radiance Fields from Sparse and Noisy Poses, https://doi.org/10.1109/cvpr52729.2023.00408, fetched 2026-10-26
