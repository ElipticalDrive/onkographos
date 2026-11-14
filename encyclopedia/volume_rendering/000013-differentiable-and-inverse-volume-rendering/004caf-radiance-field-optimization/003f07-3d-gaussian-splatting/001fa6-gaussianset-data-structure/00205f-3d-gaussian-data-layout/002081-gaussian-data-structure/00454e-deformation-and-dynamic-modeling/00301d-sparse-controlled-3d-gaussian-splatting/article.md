# Sparse-Controlled 3D Gaussian Splatting

Also called: SC-GS, Sparse-Controlled

## Overview
Sparse-Controlled 3D Gaussian Splatting (SC-GS) is a method for dynamic scene representation that enables sparse user-provided control points to edit the scene, propagating edits across time via a learned deformation prior [S1, unofficial]. It uses a small set of control points with influence weights to drive the deformation of other Gaussians, resulting in a compact and editable representation [S2, unofficial]. Unlike methods that predict deformation directly from per-Gaussian features, SC-GS links control points to time-varying 6 DoF transformations, which are locally interpolated to generate motion [S3].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of canonical 3D Gaussians $\mathcal{G}_{canon}$, a sparse set of user-provided or learned control points in canonical space, and a time parameter $t \in [0, 1]$.
*   **Outputs:** Deformed Gaussian parameters $\mathcal{G}(t)$ for rendering at time $t$.

**Data Structures:**
1.  **Canonical Gaussians:** Each Gaussian $i$ has static base parameters $(\boldsymbol{\mu}_i, \mathbf{q}_i, \mathbf{s}_i, \alpha_i, \mathbf{c}_{SH,i})$ and a learned feature vector $\mathbf{f}_i$ used for deformation prediction [S3].
2.  **Control Points:** A sparse set of points $\mathcal{C} = \{\mathbf{c}_j\}$ in canonical space. Each control point $j$ is associated with a time-varying 6 DoF transformation (position $\mathbf{p}_j(t)$ and rotation $\mathbf{R}_j(t)$) [S3].
3.  **Deformation Network:** An MLP that predicts deformation deltas based on time $t$ and location [S3].

**Steps:**
1.  **Initialize:** Learn canonical Gaussian parameters and control point positions jointly with the deformation network [S3].
2.  **Predict Control Point Motion:** For each control point $j$, predict its 6 DoF transformation at time $t$. The parameters are predicted by an MLP based on time $t$ and the control point's canonical location [S3].
3.  **Compute Influence Weights:** For each Gaussian $i$, compute influence weights from nearby control points. The sources describe this as "locally interpolated" [S3] and using "influence weights" [S2, unofficial]. The exact weighting function (e.g., inverse distance) is not specified in the provided notes [G].
4.  **Interpolate Deformation:** Combine the 6 DoF transformations of the influencing control points using the influence weights to determine the total deformation for Gaussian $i$ at time $t$. This generates the motion of dense Gaussians [S3].
5.  **Apply Deformation:** Update the canonical Gaussian parameters using the predicted deformation deltas $(\Delta\boldsymbol{\mu}_i, \Delta\mathbf{q}_i, \Delta\mathbf{s}_i)$ derived from the interpolated control point motion.
    $$ \boldsymbol{\mu}_i(t) = \boldsymbol{\mu}_i + \Delta\boldsymbol{\mu}_i $$
    $$ \mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_i \otimes \Delta\mathbf{q}_i) $$
    $$ \mathbf{s}_i(t) = \mathbf{s}_i + \Delta\mathbf{s}_i $$
6.  **Render:** Rasterize the updated Gaussians $\mathcal{G}(t)$ using standard differentiable rasterization for a given camera pose and time $t$ [S1, unofficial].

**Parameters:**
*   Number of control points: Not given in the sources.
*   MLP architecture: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Motion Complexity:** The method employs an adaptive strategy to handle motion complexity [S3].
*   **Deformation Artifacts:** An ARAP (As-Rigid-As-Possible) loss is employed to maintain local rigidity and reduce artifacts [S3].

## Operating characteristics
*   **Compactness:** Results in a compact representation due to the sparse control points [S2, unofficial].
*   **Editability:** Allows for local manipulation and editing of the dynamic scene [S3].
*   **Performance:** Not given in the sources.
*   **Strengths:** Enables large deformation, soft body simulation, and local manipulation [S3]. Produces fewer artifacts and reduces blurring compared to some mesh-based editing methods like SuGaR [S3].
*   **Limitations:** Relies on the quality and placement of sparse control points.

## Background
*   **Origin:** SC-GS is introduced as a non-mesh-based method for deforming 3D Gaussian Splatting [S3].
*   **History:** It is cited in surveys of 3D editing and dynamic 3DGS variants [S1, S2, unofficial][S3].
*   **Usage:** Used for dynamic novel view synthesis and scene editing [S3].

## Subtopics

### Deformable 3DGS
Predicts deformation using an MLP on per-Gaussian features over time, without explicit control points [S2, unofficial].

### Spacetime Gaussians (STG)
Uses temporal opacity envelopes and polynomial trajectories for each Gaussian, rendering any time slice in real time [S2, unofficial].

### GaussianFlow
Uses flow-based methods for dynamic scene representation [S2, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S3] A survey on 3D editing based on NeRF and 3DGS, https://link.springer.com/article/10.1007/s11704-025-41176-9, fetched 2026-10-28
