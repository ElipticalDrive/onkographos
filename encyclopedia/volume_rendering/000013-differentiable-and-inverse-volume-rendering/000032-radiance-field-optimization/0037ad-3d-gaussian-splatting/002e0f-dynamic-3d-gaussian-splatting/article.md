# Dynamic 3D Gaussian Splatting

Also called: 3DGS with temporal constraints, Dynamic 3DGS

## Overview
Dynamic 3D Gaussian Splatting extends static 3D Gaussian Splatting to handle time-varying scenes by maintaining a persistent set of Gaussians whose positions and orientations evolve over time. It solves the problem of temporal inconsistency and flickering inherent in per-frame independent training [S1, unofficial]. Unlike deformation-field approaches that predict offsets from a canonical space, this method explicitly tracks the time-varying parameters $(\boldsymbol{\mu}_t, q_t)$ of each Gaussian using local-rigid and As-Rigid-As-Possible (ARAP) or isometry constraints [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of images $\{I_t\}$ with known camera poses $\{c2w_t\}$ and intrinsics $K$.
*   **Outputs:** A set of $N$ Gaussians, each defined by time-varying mean position $\boldsymbol{\mu}_t \in \mathbb{R}^3$, rotation quaternion $q_t \in \mathbb{R}^4$, scale $s \in \mathbb{R}^3$ (typically static or slowly varying), opacity $\alpha$, and spherical harmonics coefficients for color.

**Data Structures:**
Each Gaussian $i$ stores:
*   $\boldsymbol{\mu}_{i,t}$: 3D position at time $t$.
*   $q_{i,t}$: Rotation quaternion at time $t$.
*   $s_i$: Scale vector (assumed constant or independently optimized per frame in baselines, but here part of the dynamic state).
*   $\alpha_i$: Opacity.
*   $c_{i,t}$: Color coefficients (SH).

**Method Steps:**
1.  **Initialization:** Initialize Gaussians from the first frame using standard 3DGS initialization (e.g., SfM points) [S1, unofficial].
2.  **Temporal Tracking:** For each subsequent frame $t$, update $\boldsymbol{\mu}_{i,t}$ and $q_{i,t}$ based on the previous frame $t-1$ and the observed motion.
3.  **Constraint Application:** Enforce physical plausibility using local-rigid and ARAP/isometry constraints. This ensures that the deformation of the Gaussian cloud respects the local geometry of the scene, preventing shearing or tearing artifacts [S1, unofficial].
4.  **Rendering:** Render the current frame using the updated Gaussians. The rendering equation remains the same as static 3DGS, but the Gaussian parameters are time-dependent:
    $$
    \hat{C}_t(\mathbf{r}) = \sum_{i \in \mathcal{N}_t} c_{i,t} \alpha_{i,t} \prod_{j=1}^{t-1} (1 - \alpha_{j,t})
    $$
    where $\mathcal{N}_t$ are the Gaussians sorted by depth for frame $t$.
5.  **Optimization:** Minimize the photometric loss $\mathcal{L}$ between $\hat{C}_t$ and $I_t$, plus a temporal regularization term derived from the ARAP/isometry constraints. Gradients flow through the time-varying parameters $\boldsymbol{\mu}_t$ and $q_t$ [S1, unofficial].

**Formulas:**
The ARAP constraint minimizes the deviation from rigid body motion for local neighborhoods of Gaussians. For a Gaussian $i$ and its neighbor $j$, the constraint encourages:
$$
\| (\boldsymbol{\mu}_{j,t} - \boldsymbol{\mu}_{i,t}) - R_t (\boldsymbol{\mu}_{j,t-1} - \boldsymbol{\mu}_{i,t-1}) \|_2^2 \approx 0
$$
where $R_t$ is the rotation matrix derived from $q_t$. The exact implementation of the ARAP energy minimization is not detailed in the sources [S1, unofficial].

**Parameters:**
*   Learning rate: Not given in the sources.
*   Constraint weights: Not given in the sources.

**Edge Cases:**
*   **Occlusion:** Gaussians may become occluded and reappear. The sources do not specify how occlusion is handled in the tracking step [S1, unofficial].
*   **Rapid Motion:** ARAP constraints may fail if motion exceeds the local neighborhood's rigid assumption.

## Subtopics

### [GaussianFlow](6149-gaussianflow.md)
A method using an explicit flow field for temporal regularization in dynamic scenes. It differs by modeling motion as a continuous flow field rather than direct Gaussian parameter tracking. Choose when explicit motion vectors are available or when flow-based regularization provides better stability than ARAP constraints. See [GaussianFlow](6149-gaussianflow.md).

### Dynamic 3D Gaussians
A method that performs tracking by persistent dynamic view synthesis, linking the motion of each Gaussian to the movement of a corresponding 3D point in the scene. It differs by explicitly associating Gaussians with tracked 3D points. Choose when point tracking is robust and when explicit point correspondence is desired. See [Dynamic 3D Gaussians](6149-gaussianflow.md) (Note: Source lists this as a minor variant but does not provide a separate link; it is a distinct approach from the main Dynamic 3DGS method).

### [GaussianFlow](6149-gaussianflow.md)
A method that accelerates deformable 3DGS by applying temporal pruning. It identifies static and active Gaussians and groups the dynamic parts by motion similarity to reduce the number of deformation queries. Choose when computational efficiency is critical and scenes have distinct static/dynamic regions. See [SpeeDe3DGS](6149-gaussianflow.md) (Note: Source lists this as a minor variant but does not provide a separate link).

## Operating characteristics
*   **Time Complexity:** Depends on the number of Gaussians $N$ and the cost of solving the ARAP/isometry constraints per frame. Generally higher than static 3DGS due to temporal optimization overhead.
*   **Memory Complexity:** $O(N)$ per frame, but requires storing history or constraints for optimization.
*   **Performance:** Real-time rendering is possible if the constraint solving is efficient. The sources do not provide specific frame rates or hardware benchmarks [S1, S2, unofficial].
*   **Strengths:** Temporal consistency, physically plausible deformations via ARAP.
*   **Limitations:** Complexity of constraint solving, potential for drift if constraints are not tight enough.

## Background
Dynamic 3D Gaussian Splatting was introduced by Luiten et al. in 3DV'24 [S1, unofficial]. It addresses the limitations of per-frame baselines which suffer from storage explosion and temporal flicker [S1, unofficial]. It is part of the broader family of 4D Gaussian Splatting methods, distinct from deformation-field approaches like 4DGS (Wu et al., CVPR'24) and Deformable 3DGS (Yang et al., CVPR'24) [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
