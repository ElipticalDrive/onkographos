# Test-time Optimization with Sim(3) Initialization

Also called: BARF test optimization, Sim(3) + opt

## Overview
A testing procedure involving Sim(3) alignment initialization followed by additional test-time optimization to adjust test poses for view synthesis [S1, unofficial]. It sets itself apart by using a global rigid-similarity transformation derived from training data to bootstrap test poses, rather than relying on local neighbor proximity or unoptimized estimated poses [S1, unofficial].

## How it works
**Inputs:**
- Estimated camera poses for training images ($P_{train}$) and ground truth poses for training images ($P_{train}^{gt}$).
- Estimated camera poses for test images ($P_{test}$).
- Photometric loss function ($\mathcal{L}_{photo}$) and a NeRF model (or similar implicit representation).

**Outputs:**
- Optimized camera poses for test images ($P_{test}^{opt}$).

**Data Structures:**
- Camera pose $P_i$ represented as a $4 \times 4$ transformation matrix or a tuple $(R_i, t_i)$ where $R_i \in SO(3)$ and $t_i \in \mathbb{R}^3$.

**Steps:**
1. **Sim(3) Initialization:** Compute a Sim(3) transformation matrix $T_{sim3}$ using Procrustes analysis on the camera pose locations (centers) of the training images. Apply this transformation to the estimated test poses:
   $$ P_{test}^{init} = T_{sim3} \cdot P_{test} $$
   This aligns the test poses to the coordinate system of the ground truth training poses [S1, unofficial].
2. **Test-time Optimization:** Optimize the parameters of $P_{test}^{init}$ by minimizing the photometric loss $\mathcal{L}_{photo}$ between rendered images and ground truth test images. The optimization updates the rotation and translation components of the test poses [S1, unofficial].

**Parameters:**
- Test-optimization iteration count: 100 iterations for methods other than NoPe-NeRF [S1, unofficial].
- Note: In some pipelines (e.g., GNeRF), test-time optimization is interleaved with training, resulting in higher total iteration counts (e.g., ~4230) [S3].

**Pitfalls:**
- Incorrect pose estimations can affect the Sim(3) alignment, causing the initialization to fail or converge to a poor local minimum [S1, unofficial].
- Procrustes alignment is sensitive to outlier images in the training set, which can cause exaggerated misalignment [S3].

## Subtopics

### SAMURAI Initialization
A variant using specific initialization schemes for direction vectors. Fixed initialization sets all initial directions to [Center, Above, Center] (North pole). When optimizing poses in SO(3), predicted camera locations are overwritten using $t_i = -R_i \times T_i$. Manual initialization allows specifying each view direction with a triplet [Left/Center/Right, Above/Center/Below, Front/Center/Back], excluding [Center, Center, Center], yielding 26 possible initial directions [S2].

### Test-time optimization for view synthesis
A variant where test-time optimization is interleaved with the training process. It registers test cameras using the Procrustes transformation from training images and optimizes test poses with photometric loss to factor out pose error effects. This approach requires more iterations (typically ~4230, or up to 1000 for scenes with outliers) to converge accurately [S3].

## Operating characteristics
- **Time Complexity:** Depends on the number of optimization iterations and the cost of rendering per iteration. Standard implementations use 100 iterations [S1, unofficial], while interleaved approaches may use thousands [S3].
- **Memory Complexity:** Low, as it only stores pose parameters and the existing NeRF model.
- **Strengths:** Effective when estimated poses can be aligned precisely to ground truth poses [S1, unofficial].
- **Limitations:** Sensitive to outliers in the training set which degrade the Procrustes alignment [S3]. Struggles with complex trajectories and reduced overlap if neighbor-based initialization is used instead (though this method uses Sim(3), not neighbor) [S1, unofficial].
- **When to use:** When global alignment of test poses to training poses is feasible and outliers are minimal.
- **When not to use:** When training data contains significant outliers that distort the Procrustes alignment [S3].

## Background
Originated from the BARF (Blending Aliased Reconstructions for Forward Modeling) approach [S1, unofficial]. The Sim(3) alignment approach is derived from the official open-source version of BARF [S1, unofficial]. It is used in evaluating novel view synthesis on datasets like NeRFBuster and Free-Dataset [S1, unofficial].

## Sources
[S1] CT-NeRF: Incremental Optimizing Neural Radiance Field and Poses with Complex Trajectory, https://doi.org/10.48550/arxiv.2404.13896, fetched 2026-10-26 (unofficial)
[S2] MELON: NeRF with Unposed Images in SO(3), https://doi.org/10.1109/3dv62453.2024.00084, fetched 2026-10-30
[S3] LU-NeRF: Scene and Pose Estimation by Synchronizing Local Unposed NeRFs, https://doi.org/10.1109/iccv51070.2023.01679, fetched 2026-10-26
