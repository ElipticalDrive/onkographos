# 4D-GS
Also called: 4D Gaussian Splatting
## Overview
4D-GS extends 3D Gaussian Splatting to dynamic scenes by parameterizing Gaussian attributes as functions of time, enabling the reconstruction and rendering of non-rigidly deforming objects from multi-view video [S1, unofficial]. It distinguishes itself from other dynamic Gaussian methods by replacing standard positional encoding with multi-resolution hexplane voxels for spatiotemporal information and employing a two-stage optimization process that first stabilizes the static scene before learning the deformation field [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of multi-view images $\{I_v\}$ with known camera poses and timestamps $t$.
*   **Outputs:** A set of 4D Gaussians, where each Gaussian is defined by a base set of static attributes and a time-dependent deformation field.

**Data Structures:**
1.  **Static Gaussians:** Each Gaussian $i$ stores base attributes: position $\mathbf{c}_i$, rotation quaternion $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_{i, SH}$.
2.  **Hexplane Voxels:** A multi-resolution structure encoding spatiotemporal information. It consists of six 2D planes (hexplanes) representing the 2D projections of the 3D space and time dimension. Each plane is a grid of learnable features.
3.  **Attribute MLPs:** Compact Multi-Layer Perceptrons (MLPs) dedicated to predicting delta values for specific attributes (e.g., position, rotation, scale) based on inputs from the hexplane voxels and the query time $t$.

**Algorithm Steps:**
1.  **Initialization:** Initialize a static 3D Gaussian point cloud using standard 3DGS initialization (e.g., from SfM points).
2.  **Stage 1: Static Optimization:** Optimize the static Gaussian attributes ($\mathbf{c}_i, \mathbf{q}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_{i, SH}$) using standard 3DGS rendering and photometric loss until convergence. This establishes a stable baseline for the scene [S1, unofficial].
3.  **Stage 2: Deformation Field Learning:**
    *   **Voxel Encoding:** For each Gaussian $i$ and query time $t$, encode its spatial position $\mathbf{c}_i$ and time $t$ into the multi-resolution hexplane voxels. This replaces positional encoding [S1, unofficial].
    *   **Temporal Smoothing:** Add linearly decaying Gaussian noise to the encoded time vector to improve temporal smoothing without additional computational overhead [S1, unofficial].
    *   **Delta Prediction:** Pass the encoded spatiotemporal features through attribute-specific compact MLPs to predict delta values ($\Delta \mathbf{c}, \Delta \mathbf{q}, \Delta \mathbf{s}$, etc.) [S1, unofficial].
    *   **Attribute Update:** Compute the dynamic attributes for time $t$:
        $$ \mathbf{c}_i(t) = \mathbf{c}_i + \Delta \mathbf{c}_i(t) $$
        $$ \mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_i + \Delta \mathbf{q}_i(t)) $$
        $$ \mathbf{s}_i(t) = \mathbf{s}_i + \Delta \mathbf{s}_i(t) $$
    *   **Rendering:** Render the scene using the time-dependent attributes $\mathbf{c}_i(t), \mathbf{q}_i(t), \mathbf{s}_i(t)$ for the camera timestamp $t$.
    *   **Optimization:** Minimize photometric loss (L1) between rendered and ground truth images. Optimize the MLP weights and the static Gaussian attributes jointly [S1, unofficial].

**Parameters:**
*   **Hexplane Resolution:** Not given in the sources.
*   **MLP Architecture:** Compact MLPs; specific layer sizes not given in the sources.
*   **Noise Schedule:** Linearly decaying Gaussian noise on the time vector; decay rate not given in the sources.

**Edge Cases and Pitfalls:**
*   **Temporal Artifacts:** Without noise injection, time interpolation can amplify artifacts [S1, unofficial].
*   **Optimization Stability:** Jointly optimizing static and dynamic attributes from scratch can be unstable; the two-stage approach mitigates this [S1, unofficial].
## Subtopics
## Operating characteristics
*   **Complexity:** Rendering complexity is similar to 3DGS, but training involves additional overhead from hexplane voxel lookups and MLP inference for each Gaussian at each time step.
*   **Performance:** Not given in the sources.
*   **Strengths:** Stable training via two-stage optimization; efficient spatiotemporal encoding via hexplanes; robust temporal smoothing via noise injection [S1, unofficial].
*   **Limitations:** Requires a static baseline; performance on highly dynamic scenes with large non-rigid deformations may be limited by the capacity of compact MLPs [S1, unofficial].
## Background
4D-GS was introduced as an extension of 3D Gaussian Splatting to handle dynamic scenes. It builds on the principles of temporal parameterization and canonical space mapping shared by its siblings [S1, unofficial]. The method was developed to address the instability of directly optimizing dynamic Gaussians and the inefficiency of positional encoding for spatiotemporal data.
## Sources
[S1] Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
