# Warped Views
Also called: Angle-based view warping, Rodrigues rotation sampling

## Overview
Warped Views is a view sampling strategy that generates novel viewpoints by rotating training cameras around an estimated scene center axis. Unlike SDS sampled views, which may be too distant to provide sufficient overlap, Warped Views ensures large view overlapping by applying small, predefined angular perturbations to the original camera poses. It is primarily used in conjunction with Depth Warping Loss to enforce geometric consistency between slightly different perspectives of the same scene.

## How it works
**Inputs:** A set of training camera poses, where each pose $P_i$ consists of a position and a rotation matrix $R_i$.
**Outputs:** A set of warped camera poses.

**Data Structures:**
- Camera Pose $P$: A 3D vector representing the camera position.
- Rotation Matrix $R$: A $3 \times 3$ matrix. The up-vector is extracted from the first column: $Y = [R_{11}, R_{21}, R_{31}]^T$.

**Steps:**
1. **Extract Up-Vectors:** For each training camera $i$, extract the up-vector $Y_i$ from the first column of its rotation matrix $R_i$.
2. **Compute Average Up-Vector:** Calculate the mean of all up-vectors and normalize it to obtain the estimated center axis $\bar{Y}$:
   $$ \bar{Y} = \frac{1}{N} \sum_{i=1}^{N} Y_i $$
   $$ \bar{Y} = \frac{\bar{Y}}{\|\bar{Y}\|} $$
   [G] If $\|\bar{Y}\|$ is near zero, the axis is undefined; in practice, a default up-vector (e.g., $[0,1,0]$) is used.
3. **Define Rotation Angles:** Select a set of small angles $\theta$. Typical values are $\{-3^\circ, -1.5^\circ, 1.5^\circ, 3^\circ\}$ [S1].
4. **Rotate Camera Poses:** For each training camera position $P$ and each angle $\theta$, compute the new position $P'$ using Rodrigues' rotation formula around the axis $\bar{Y}$:
   $$ P' = P \cos \theta + (P \times \bar{Y}) \sin \theta + \bar{Y} (\bar{Y} \cdot P) (1 - \cos \theta) $$
   [G] The rotation is applied to the position vector $P$ relative to the origin. The camera orientation should also be rotated by $\theta$ around $\bar{Y}$ to maintain the correct look-at direction, though the source primarily specifies the position warp.
5. **Generate Views:** Render the scene from each new position $P'$ with the corresponding rotated orientation.

**Parameters:**
- Rotation angles $\theta$: $-3^\circ, -1.5^\circ, 1.5^\circ, 3^\circ$ [S1].
- Number of warped views per training camera: 4 [S1].

**Edge Cases:**
- **Degenerate Axis:** If the average up-vector $\bar{Y}$ has a norm close to zero, the rotation axis is ill-defined. This may occur if training cameras are distributed symmetrically around the vertical axis.
- **Precision:** Ensure angles are converted to radians before trigonometric calculations.

## Subtopics

### Extrapolated View Generation
Synthesizes novel views by perturbing the original camera center to maintain scene overlap while providing a novel perspective. It perturbs the camera center by $\pm 0.25$ units along the x and y axes and translates it backward by $0.5$ units along the z-axis. This backward-only perturbation ensures high visibility overlap with the original training view, keeping most scene content within shared frustums for reliable correspondence estimation (e.g., for SA-Point generation) [S2].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of training cameras and the number of sampled angles.
- **Memory Complexity:** Low; only requires storing the additional camera poses.
- **Strengths:** Provides high view overlap compared to distant sampling strategies like SDS, which is crucial for depth warping and consistency losses.
- **Limitations:** The overlap is limited by the small rotation angles; it does not cover wide baselines.
- **Usage:** Applied to sparse input views in conjunction with Depth Warping Loss [S1].

## Background
Warped Views were introduced in the context of SparseGS (Sparse View Synthesis Using 3D Gaussian Splatting) to address the lack of view overlap when using standard SDS sampling for depth consistency losses [S1]. The method leverages Rodrigues' rotation formula to create small perturbations around the scene center, estimated from the average up-vector of the training cameras.

## Sources
[S1] Sparsegs: Sparse View Synthesis Using 3D Gaussian Splatting, https://doi.org/10.1109/3dv66043.2025.00100, fetched 2026-10-26
[S2] SA-ResGS: Self-augmented Residual 3D Gaussian Splatting for Next Best View Selection, https://doi.org/10.1007/978-3-032-37013-6_17, fetched 2026-10-29
