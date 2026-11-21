# Normal Supervision Loss

Also called: Lnormal, Monocular Normal Supervision, Normal Loss with Geometric Prior

## Overview
Normal Supervision Loss guides surface orientation by comparing volume-rendered normal maps against geometric priors derived from pretrained monocular depth/normal estimation networks (e.g., PI3 or Omnidata) [S1, unofficial]. It solves the problem of noisy or ambiguous surface normals in sparse-view or reflective scenarios by providing explicit geometric constraints [S1, unofficial]. Unlike sibling methods that rely on gradients of rendered depths, which are unstable, this method uses direct normal alignment [S1, unofficial]. It is distinct from [Geometric Prior Guided Geometry Learning](16865-geometric-prior-guided-geometry-learning.md) in its specific formulation of rendering voxel normals and its annealing schedule [S1, unofficial].

## How it works
**Inputs:**
- Volume representation (e.g., trilinearly interpolated grid or SDF field).
- Camera parameters for each view $i$.
- Pretrained monocular network outputs: per-view point maps $P_i$ or normal priors.
- Image mask $M$ defining valid pixels.

**Outputs:**
- Scalar loss value $L_{normal}$.

**Data Structures:**
- **Voxel Normal ($n_v$):** Computed analytically from the interpolation function. For a voxel $v$ with geometry $g_{ov}$, the normal at the voxel center $q_c = (0.5, 0.5, 0.5)$ is:
  $$ n_v = \text{normalize}(\nabla_q \text{interp}(g_{ov}, q_c)) $$
  where $\nabla_q$ is the gradient with respect to local voxel coordinates and $\text{normalize}(\cdot)$ is L2 normalization [S1, unofficial].
- **Rendered Normal Map ($N_{rendered}$):** Accumulated along rays using the volumetric rendering equation, replacing color $c_i$ with normal $n_i$:
  $$ N_{rendered}(u) = \sum_{i=1}^{N_{ray}} T_i \alpha_i n_i $$
  where $T_i$ is transmittance, $\alpha_i$ is opacity, and $n_i$ is the interpolated normal at sample point $i$ [S1, unofficial].
- **Prior Normal Map ($N_{prior}$):** Derived from the per-view point map $P_i$ by computing surface normals from the 3D points projected to 2D [S1, unofficial].

**Steps:**
1. **Compute Prior Normals:** For each camera view $i$, generate $N_{prior}$ from the monocular point map $P_i$ [S1, unofficial].
2. **Compute Voxel Normals:** Calculate $n_v$ for all voxels using the analytical gradient of the trilinear interpolation at the voxel center [S1, unofficial].
3. **Render Normals:** For each pixel $u$ in the image, render $N_{rendered}(u)$ by accumulating voxel normals along the ray using the standard alpha-compositing formula [S1, unofficial].
4. **Transform Coordinates:** Ensure $N_{rendered}$ and $N_{prior}$ are in the same coordinate system (e.g., world or camera space) [S2].
5. **Compute Loss:** Calculate the robust cosine distance loss over valid pixels $u \in M$:
   $$ L_{normal} = \frac{1}{|M|} \sum_{u \in M} \left( 1 - N_{rendered}(u) \cdot N_{prior}(u) \right) $$
   Note: Source [S2] defines a slightly different formulation using L1 norms for both vector difference and cosine discrepancy:
   $$ L_{normal} = \sum_{r \in R} \| \hat{N}(r) - \bar{N}(r) \|_1 + \| 1 - \hat{N}(r)^T \bar{N}(r) \|_1 $$
   where $\hat{N}$ is rendered and $\bar{N}$ is prior [S2]. The SVRecon implementation [S1, unofficial] uses the cosine distance form. Implementers should verify which variant matches their source codebase.
6. **Apply Weight Schedule:** Multiply $L_{normal}$ by $\lambda_n$ based on iteration $\tau$ [S1, unofficial].

**Parameters:**
- **DTU Scenes:**
  - $\lambda_n = 0.10$ for $0 \le \tau < 4000$
  - $\lambda_n = 0.01$ for $4000 \le \tau < 6000$
  - $\lambda_n = 0.00$ otherwise [S1, unofficial]
- **Tanks-and-Temples Scenes:**
  - $\lambda_n = 0.01$ for $0 \le \tau < 4000$
  - $\lambda_n = 0.005$ for $4000 \le \tau < 8000$
  - $\lambda_n = 0.00$ otherwise [S1, unofficial]
- **MonoSDF Default:** $\lambda_3 = 0.05$ (constant) [S2].

**Edge Cases and Pitfalls:**
- **Noisy Priors:** $N_{prior}$ is derived from imperfect monocular estimates and can be noisy [S1, unofficial]. This is mitigated by annealing $\lambda_n$ to zero in final fine-tuning stages [S1, unofficial].
- **Reflective Objects:** Standard training may fail on reflective surfaces (e.g., DTU scan 110) due to specular highlights causing noisy geometry. Extending normal supervision until the end of training reduces floating artifacts and improves mesh smoothness [S1, unofficial].
- **Coordinate Systems:** Failure to transform $N_{prior}$ and $N_{rendered}$ to the same coordinate system results in incorrect loss computation [S2].

## Operating characteristics
- **Time Complexity:** Adds $O(N_{rays} \cdot N_{samples})$ per iteration for normal rendering, similar to color rendering. Prior normal computation is $O(N_{pixels})$ per view [S1, unofficial][S2].
- **Memory Complexity:** Requires storing $N_{rendered}$ and $N_{prior}$ maps. No additional large structures if priors are computed on-the-fly or cached per view [S1, unofficial].
- **Strengths:** Provides strong geometric guidance for surface orientation, especially in sparse-view settings [S1, unofficial]. Effective at reducing floating artifacts on reflective surfaces [S1, unofficial].
- **Limitations:** Dependent on the quality of the monocular prior network [S1, unofficial]. Noisy priors can mislead optimization if not annealed [S1, unofficial].
- **When to Use:** Sparse-view reconstruction, reflective objects, or when depth gradients are unreliable [S1, unofficial].
- **When Not to Use:** Dense-view scenarios with high-quality depth sensors where internal consistency losses suffice [S1, unofficial].

## Background
Originated in SVRecon [S1, unofficial] and MonoSDF [S2] as a method to leverage monocular geometric cues for neural implicit surface reconstruction. SVRecon introduced the use of PI3 point maps for normal priors with a specific annealing schedule [S1, unofficial]. MonoSDF proposed a similar concept using angular and L1 losses [S2]. The method addresses the limitation of relying solely on photometric and depth losses, which can be ambiguous for surface orientation [S1, unofficial].

## Sources
[S1] SVRecon: Sparse Voxel Rasterization for Surface Reconstruction, https://doi.org/10.48550/arxiv.2511.17364, fetched 2026-10-26 (unofficial)
[S2] MonoSDF: Exploring Monocular Geometric Cues for Neural Implicit Surface Reconstruction, https://doi.org/10.52202/068431-1814, fetched 2026-10-26
