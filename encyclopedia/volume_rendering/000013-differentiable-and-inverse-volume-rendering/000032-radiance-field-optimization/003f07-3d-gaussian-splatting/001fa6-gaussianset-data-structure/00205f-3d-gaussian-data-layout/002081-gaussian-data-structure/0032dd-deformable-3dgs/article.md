# Deformable 3DGS

Also called: Dynamic 3DGS

## Overview
Deformable 3DGS is a method for rendering dynamic scenes by applying an MLP deformation network to per-Gaussian features over time [S1, unofficial]. It solves the problem of representing non-rigid motion in radiance fields by decomposing the scene into a static canonical representation and a time-dependent deformation field [S1, unofficial]. Unlike variants that use compact parametric encoders like HexPlane or k-Plane, Deformable 3DGS relies on a learnable MLP to predict deformation parameters directly from Gaussian attributes [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians defined in a canonical space (position $\boldsymbol{\mu}$, rotation $\mathbf{q}$, scale $\mathbf{s}$, opacity $\alpha$, spherical harmonics coefficients $\mathbf{c}$), a time parameter $t$, and an MLP deformation network $\mathcal{M}$.
*   **Outputs:** Transformed Gaussian parameters $(\boldsymbol{\mu}', \mathbf{q}', \mathbf{s}', \alpha', \mathbf{c}')$ suitable for rasterization at time $t$.

**Data Structures:**
The method extends the standard Gaussian Data Structure [2273-gaussian-data-structure.md] by storing per-Gaussian features that serve as input to the deformation network. The MLP $\mathcal{M}$ takes these features and the time $t$ as input.

**Steps:**
1.  **Feature Extraction:** For each Gaussian $i$, extract its canonical attributes $\mathbf{f}_i = [\boldsymbol{\mu}_i, \mathbf{q}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_i]$ and the current time $t$.
2.  **Deformation Prediction:** Pass $\mathbf{f}_i$ and $t$ through the MLP deformation network $\mathcal{M}$ to predict the deformation deltas:
    $$ [\Delta\boldsymbol{\mu}_i, \Delta\mathbf{q}_i, \Delta\mathbf{s}_i] = \mathcal{M}(\mathbf{f}_i, t) $$
    [G] The MLP typically outputs raw deltas for position and scale, and a quaternion offset for rotation.
3.  **Parameter Update:** Apply the predicted deltas to the canonical parameters to obtain the deformed parameters:
    $$ \boldsymbol{\mu}_i(t) = \boldsymbol{\mu}_i + \Delta\boldsymbol{\mu}_i $$
    $$ \mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_i + \Delta\mathbf{q}_i) $$
    $$ \mathbf{s}_i(t) = \mathbf{s}_i + \Delta\mathbf{s}_i $$
    [G] Opacity $\alpha$ and SH coefficients $\mathbf{c}$ are typically kept static or optionally modulated if the network outputs additional channels.
4.  **Rasterization:** Render the deformed Gaussians using the standard differentiable rasterizer [3D Gaussian Splatting](52-name.md).

**Parameters:**
*   **MLP Architecture:** Not given in the sources.
*   **Time $t$:** Normalized to $[0, 1]$ or absolute timestamp depending on implementation.

**Edge Cases and Pitfalls:**
*   **Temporal Consistency:** The MLP may produce discontinuous deformations between frames if not regularized.
*   **Precision:** Quaternion normalization is critical to maintain valid rotations after additive updates.

## Subtopics

### SplineGS
SplineGS replaces the discrete per-frame MLP deformation with a smooth spline model for the trajectory of each primitive's center and rotation as a function of time [S2, unofficial]. It ensures physically plausible, smooth motion with fewer parameters by modeling the position $\mu_i(t)$ as:
$$ \mu_i(t) = \sum_{k=0}^K \mathbf{B}_k(t) \mathbf{p}_{i,k} $$
where $\mathbf{B}_k$ are B-spline basis functions and $\mathbf{p}_{i,k}$ are control points [S2, unofficial]. This approach is chosen when smooth, continuous motion is required and parameter efficiency is prioritized over the flexibility of an MLP [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the MLP evaluation per Gaussian per frame, plus standard rasterization. The MLP adds significant overhead compared to static 3DGS.
*   **Memory Complexity:** Requires storing the MLP weights and per-Gaussian features. No additional large buffers compared to static 3DGS.
*   **Strengths:** Flexible representation of complex non-rigid deformations; end-to-end differentiable.
*   **Limitations:** Slower rendering due to MLP inference; potential for temporal artifacts if the MLP is not well-regularized.
*   **When to use:** Dynamic scenes with complex, non-rigid motion where compact parametric models (like splines) are insufficient.

## Background
Deformable 3DGS was introduced by Yang et al. in CVPR'24 [S1, unofficial]. It builds on the static 3D Gaussian Splatting framework [3D Gaussian Splatting](52-name.md) by adding a deformation network to handle temporal dynamics. It is part of the broader family of 4D Gaussian Splatting methods, which aim to extend real-time rendering to dynamic scenes.

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
