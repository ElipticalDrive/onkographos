# Deformable 3DGS

Also called: Dynamic 3DGS

## Overview
Deformable 3DGS is a method for rendering dynamic scenes by applying a Multi-Layer Perceptron (MLP) deformation network to per-Gaussian features over time [S1, unofficial]. It solves the problem of representing non-rigid motion in 3D Gaussian Splatting by decomposing the scene into a static canonical representation and a time-dependent deformation field [S1, unofficial]. Unlike sibling methods that use compact geometric encoders like HexPlane or k-Plane to predict deformations, Deformable 3DGS relies on a learned MLP to map canonical features and time to displacement, rotation, and scale updates [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of canonical 3D Gaussians (defined in the parent article [Gaussian Data Structure](2273-gaussian-data-structure.md)), a time parameter $t$, and an MLP deformation network.
*   **Outputs:** Time-varying Gaussian parameters (position $\boldsymbol{\mu}(t)$, rotation $\mathbf{q}(t)$, scale $\mathbf{s}(t)$, opacity $\alpha$, and spherical harmonic coefficients) for rendering at time $t$.

**Data Structures:**
The method extends the standard Gaussian data structure by storing canonical parameters and associating each Gaussian with an MLP. The MLP takes as input the canonical features (e.g., position, rotation, scale, SH coefficients) concatenated with the time $t$, and outputs the deltas $(\Delta\boldsymbol{\mu}, \Delta\mathbf{q}, \Delta\mathbf{s})$ [S1, unofficial].

**Algorithm:**
1.  **Canonical Representation:** Initialize a set of 3D Gaussians in a canonical (static) space. These Gaussians represent the scene at a reference time or an average state.
2.  **Deformation Network:** Train an MLP $\mathcal{F}_{\theta}$ that predicts the deformation for each Gaussian. The input to $\mathcal{F}_{\theta}$ is the concatenation of the Gaussian's canonical attributes and the current time $t$.
3.  **Time-Varying Parameter Prediction:** For a given time $t$ and a Gaussian $i$ with canonical attributes $\mathbf{c}_i$:
    *   Compute the deformation deltas:
        $$ [\Delta\boldsymbol{\mu}_i, \Delta\mathbf{q}_i, \Delta\mathbf{s}_i] = \mathcal{F}_{\theta}(\mathbf{c}_i, t) $$
    *   Update the Gaussian parameters:
        $$ \boldsymbol{\mu}_i(t) = \boldsymbol{\mu}_i + \Delta\boldsymbol{\mu}_i $$
        $$ \mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_i + \Delta\mathbf{q}_i) $$
        $$ \mathbf{s}_i(t) = \mathbf{s}_i + \Delta\mathbf{s}_i $$
    *   Note: The exact formulation of how $\Delta\mathbf{q}$ is applied (addition vs. quaternion multiplication) is not explicitly detailed in the sources, but standard practice involves quaternion normalization [G].
4.  **Rendering:** Render the time-varying Gaussians using the standard 3DGS rasterization pipeline [S1, unofficial].

**Parameters:**
*   **MLP Architecture:** Not given in the sources.
*   **Time Input:** Typically normalized to $[0, 1]$ or absolute timestamp.

**Edge Cases and Pitfalls:**
*   **Temporal Consistency:** The MLP must ensure smooth transitions between frames. Discontinuities in the MLP output can cause flickering [S1, unofficial].
*   **Overfitting:** The MLP may overfit to specific time steps if not regularized, leading to poor interpolation [G].

## Subtopics

### SplineGS
A method that replaces the discrete per-frame deformation with a smooth spline modeling the trajectory of each primitive's center and rotation as a function of time [S2, unofficial]. It ensures physically plausible, smooth motion with fewer parameters by using B-spline basis functions [S2, unofficial]. The position $\mu_i(t)$ is calculated as:
$$ \mu_i(t) = \sum_{k=0}^K \mathbf{B}_k(t) \mathbf{p}_{i,k} $$
where $\mathbf{B}_k$ are B-spline basis functions and $\mathbf{p}_{i,k}$ are control points [S2, unofficial]. Choose this when smooth, continuous motion is required and parameter efficiency is critical [S2, unofficial].

### VolSegGS
A method for segmenting and tracking structures across timesteps in dynamic volumes by representing scenes as deformable 3D Gaussians, sidestepping grid memory constraints [S3, unofficial]. It performs coarse-level segmentation by clustering Gaussians by view-independent colors, refines clusters into fine-level segments using an affinity field network guided by SAM 2D prompts, and maintains segments across timesteps through the deformation field [S3, unofficial]. It achieved PSNR values of 25.76–29.02 dB at 88–89 FPS across five time-varying datasets at resolutions up to 960x1440x240 [S3, unofficial]. Choose this for dynamic volumetric segmentation tasks [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering time is similar to standard 3DGS, as the deformation is computed per-Gaussian before rasterization. The MLP inference adds a small constant overhead per Gaussian [S1, unofficial].
*   **Memory Complexity:** Higher than static 3DGS due to the storage of the MLP weights and the need to store canonical parameters [S1, unofficial].
*   **Performance:** Real-time rendering is achievable, but depends on the MLP size and the number of Gaussians [S1, unofficial].
*   **Strengths:** Handles complex non-rigid deformations; flexible representation [S1, unofficial].
*   **Limitations:** Requires training an MLP; potential for temporal artifacts if not regularized; higher memory usage than static methods [S1, unofficial].

## Background
Deformable 3DGS was introduced by Yang et al. in CVPR'24 [S1, unofficial]. It is part of the broader effort to extend 3D Gaussian Splatting to dynamic scenes, competing with methods like 4DGS which use HexPlane/k-Plane decoders [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S3] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
