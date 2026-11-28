# Deformable 3DGS

Also called: 4D Gaussian Splatting, Dynamic 3DGS

## Overview
Deformable 3DGS extends static 3D Gaussian Splatting to dynamic scenes by modeling time-dependent geometric and appearance parameters [S1, S2, unofficial]. It solves the problem of reconstructing and rendering moving subjects (e.g., people, fluids) from multi-view video by decomposing the 4D representation into a static canonical set of Gaussians and a deformation function that predicts their state at any time $t$ [S2, unofficial]. It distinguishes itself from sibling methods like Sparse-Controlled 3D Gaussian Splatting by using a direct MLP-based deformation network on per-Gaussian features rather than sparse control points, and from HexPlane-based variants by its specific architectural choice for feature-to-deformation mapping [S2, unofficial].

## How it works
**Inputs:** Multi-view video frames with known camera poses and timestamps $t \in [0, 1]$.
**Outputs:** A set of time-varying 3D Gaussians $\mathcal{G}(t)$ rendered into images.

**Data Structures:**
1.  **Canonical Gaussians $\mathcal{G}_{canon}$:** Each Gaussian $i$ stores static base parameters:
    *   Mean $\boldsymbol{\mu}_i \in \mathbb{R}^3$
    *   Rotation $\mathbf{q}_i \in \mathbb{R}^4$ (unit quaternion)
    *   Scale $\mathbf{s}_i \in \mathbb{R}^3$
    *   Opacity $\alpha_i \in \mathbb{R}$
    *   Spherical Harmonics coefficients $\mathbf{c}_{SH,i}$
2.  **Deformation Network $f_\theta$:** An MLP that takes canonical features and time $t$ as input.
    *   Input: Concatenation of per-Gaussian latent features (or position) and time $t$.
    *   Output: Deltas $\Delta\boldsymbol{\mu}_i, \Delta\mathbf{q}_i, \Delta\mathbf{s}_i$.

**Algorithm:**
1.  **Initialization:** Initialize $\mathcal{G}_{canon}$ using standard 3DGS initialization from static reconstruction or SfM points [S1, unofficial].
2.  **Deformation Prediction:** For each Gaussian $i$ and time $t$:
    $$ \Delta\boldsymbol{\mu}_i = f_\theta(\boldsymbol{\mu}_i, t) $$
    $$ \Delta\mathbf{q}_i = f_\theta(\mathbf{q}_i, t) $$
    $$ \Delta\mathbf{s}_i = f_\theta(\mathbf{s}_i, t) $$
    *Note: Specific implementations may use different input features (e.g., latent codes $l$) [S4].*
3.  **Time-Varying Parameter Update:**
    $$ \boldsymbol{\mu}_i(t) = \boldsymbol{\mu}_i + \Delta\boldsymbol{\mu}_i $$
    $$ \mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_i \otimes \Delta\mathbf{q}_i) $$
    $$ \mathbf{s}_i(t) = \mathbf{s}_i + \Delta\mathbf{s}_i $$
    where $\otimes$ is quaternion multiplication [S1, unofficial].
4.  **Rendering:** Rasterize $\mathcal{G}(t)$ using standard differentiable rasterization with camera pose and time $t$ [S1, unofficial].
5.  **Optimization:** Minimize photometric loss $\mathcal{L} = \mathcal{L}_{rec} + \lambda \mathcal{L}_{reg}$, updating canonical parameters, features, and network weights via SGD [S1, unofficial].

**Parameters:**
*   Learning rate: Not given in the sources.
*   $\lambda$ (regularization weight): Not given in the sources.

**Edge Cases:**
*   Quaternion normalization prevents drift in rotation updates [G].
*   Precision issues may arise if $\Delta\mathbf{s}_i$ causes negative scales; clipping or absolute value may be required [G].

## Subtopics

### SplineGS
A method that replaces discrete per-frame deformation with a smooth spline modeling the trajectory of each primitive's center and rotation as a function of time, ensuring physically plausible, smooth motion with fewer parameters [S3, unofficial]. It uses B-spline basis functions $\mathbf{B}_k(t)$ and control points $\mathbf{p}_{i,k}$:
$$ \mu_i(t) = \sum_{k=0}^K \mathbf{B}_k(t) \mathbf{p}_{i,k} $$
[S3, unofficial].

### MLP Architecture for Local Deformation
A neural network that takes local body pose and latent codes to predict residual transformations (translation, rotation) and shading for Gaussians [S4]. Input is the concatenation of local body pose vector $\theta_t$ and per-Gaussian learnable latent features $l$ [S4]. Hidden layers: 2 layers, 64 neurons each, ReLU activations [S4]. Output heads: 3 separate heads, each with 2 layers, outputting: 1) Translation vector $t_{mlp}$, 2) Rotation quaternion $q_{mlp}$, 3) Scaling factor $s$ [S4]. Using latent codes $l$ instead of position leads to better performance [S4]. The final observed position and orientation are computed as:
$$ \mathbf {p}_{\text {obs}} = \mathbf {p}_{\text {lbs}} + (\mathbf {R}_{t}\mathbf {t}_{\text {mlp}}) $$
$$ \mathbf {q}_{\text {obs}} = \mathbf {q}_{\text {mlp}} \circ \mathbf {q}_{\text {lbs}} $$
[S4].

## Operating characteristics
*   **Time Complexity:** Depends on the deformation network size and number of Gaussians. Real-time rendering is feasible if the deformation network is lightweight [S3, unofficial].
*   **Memory Complexity:** Stores canonical Gaussians plus deformation network weights. More compact than storing full 4D Gaussians for every frame [S2, unofficial].
*   **Strengths:** Handles dynamic scenes (people moving, fluids) [S1, unofficial]. Can achieve real-time rendering [S3, unofficial].
*   **Limitations:** Performance depends on the deformation network's capacity. May struggle with complex, non-rigid deformations if the network is too simple [S3, unofficial].
*   **When to use:** Dynamic scene reconstruction and rendering [S1, unofficial].
*   **When not to use:** Static scenes (use standard 3DGS) or scenes requiring explicit control via sparse points (use SC-GS) [S2, unofficial].

## Background
Introduced by Yang et al. in CVPR'24 [S2, unofficial]. It is part of the broader effort to extend 3D Gaussian Splatting to 4D (spatiotemporal) representations [S2, unofficial].

## Sources
[S1] Neural Rendering — Technologies & Tricks, https://www.chengjianglong.com/ml/3D-Neural_Rendering_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S4] Human Gaussian Splatting: Real-Time Rendering of Animatable Avatars, https://doi.org/10.1109/cvpr52733.2024.00081, fetched 2026-10-26
