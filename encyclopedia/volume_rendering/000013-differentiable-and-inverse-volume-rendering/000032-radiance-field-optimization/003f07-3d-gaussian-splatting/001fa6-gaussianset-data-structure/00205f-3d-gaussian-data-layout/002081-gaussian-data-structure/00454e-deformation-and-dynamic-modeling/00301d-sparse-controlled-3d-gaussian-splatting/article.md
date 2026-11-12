# Sparse-Controlled 3D Gaussian Splatting
Also called: SC-GS, Sparse-Controlled

## Overview
Sparse-Controlled 3D Gaussian Splatting (SC-GS) is a dynamic scene representation method that enables sparse user-provided control points to edit a dynamic scene, propagating these edits across time via a learned deformation prior [S1, unofficial][S2, unofficial]. It distinguishes itself from sibling methods like Deformable 3DGS by using a small set of control points with influence weights to drive the deformation of other Gaussians, resulting in a more compact and editable representation [S2, unofficial].

## How it works
**Inputs:**
- Canonical 3D Gaussian set $\mathcal{G}_{canon}$ (position $\boldsymbol{\mu}_i$, rotation $\mathbf{q}_i$, scale $\mathbf{s}_i$, opacity $\alpha_i$, SH coefficients $\mathbf{c}_{SH,i}$) [G].
- A small set of sparse user-provided control points $\mathcal{C} = \{c_j\}_{j=1}^M$.
- Time parameter $t \in [0, 1]$.

**Outputs:**
- Deformation deltas $(\Delta\boldsymbol{\mu}_i, \Delta\mathbf{q}_i, \Delta\mathbf{s}_i)$ for each Gaussian $i$.
- Rendered image at time $t$.

**Data Structures:**
- Control Points: A list of $M$ points, each with a 3D position $\mathbf{p}_j$ and potentially an associated influence radius or weight function parameters.
- Influence Weights: For each Gaussian $i$, a weight vector $\mathbf{w}_i \in \mathbb{R}^M$ representing the influence of each control point.

**Steps:**
1. **Compute Influence Weights:** For each Gaussian $i$, calculate its influence weights $\mathbf{w}_i$ based on its spatial distance to the control points $\mathcal{C}$. A common approach is using a radial basis function (e.g., Gaussian kernel) centered at each control point $c_j$:
   $$ w_{i,j} = \exp\left(-\frac{\|\boldsymbol{\mu}_i - \mathbf{p}_j\|^2}{2\sigma_j^2}\right) $$
   where $\sigma_j$ is the influence radius of control point $j$. Normalize weights if necessary: $\hat{w}_{i,j} = \frac{w_{i,j}}{\sum_k w_{i,k}}$. [G]

2. **Predict Deformation Deltas:** Use a learned deformation network (e.g., MLP) conditioned on the normalized influence weights $\hat{\mathbf{w}}_i$ and time $t$ to predict the deformation deltas for Gaussian $i$:
   $$ (\Delta\boldsymbol{\mu}_i, \Delta\mathbf{q}_i, \Delta\mathbf{s}_i) = \text{DeformNet}(\hat{\mathbf{w}}_i, t) $$
   The network maps the sparse control influence to dense Gaussian deformation. [S2, unofficial]

3. **Apply Deformation:** Update the canonical Gaussian parameters using the predicted deltas, following the shared principles of the parent method:
   $$ \boldsymbol{\mu}_i(t) = \boldsymbol{\mu}_i + \Delta\boldsymbol{\mu}_i $$
   $$ \mathbf{q}_i(t) = \text{Normalize}(\mathbf{q}_i \otimes \Delta\mathbf{q}_i) $$
   $$ \mathbf{s}_i(t) = \mathbf{s}_i + \Delta\mathbf{s}_i $$
   [G]

4. **Render:** Rasterize the deformed Gaussians $\mathcal{G}(t)$ using standard differentiable rasterization for the given camera pose and time $t$ [S1, unofficial][G].

**Parameters:**
- Number of control points $M$: Small, user-defined.
- Influence radius $\sigma_j$: Controls the spatial extent of each control point's influence.
- Deformation Network Architecture: Typically a small MLP.

**Edge Cases and Pitfalls:**
- **Degenerate Control Points:** If control points are too close or coincident, influence weights may become unstable. [G]
- **Zero Influence:** If a Gaussian is far from all control points, weights may be near zero, leading to no deformation. Ensure network handles this gracefully. [G]
- **Precision:** Quaternion normalization is critical to prevent rotation drift. [G]

## Subtopics

## Operating characteristics
**Time Complexity:** Deformation prediction is $O(N \cdot M)$ for $N$ Gaussians and $M$ control points, assuming constant-time weight computation and network evaluation. Rendering remains $O(N)$ per pixel.
**Memory Complexity:** $O(N + M)$, significantly lower than methods storing per-Gaussian deformation networks or trajectories, as deformation is driven by sparse controls.
**Strengths:** High editability via sparse controls; compact representation; efficient deformation prediction.
**Limitations:** Quality depends on control point placement; may struggle with complex, non-local deformations if controls are insufficient.
**When to Use:** When user interaction/editing of dynamic scenes is required, or when memory efficiency is critical.
**When Not to Use:** For highly complex, autonomous dynamic scenes without user control points.

## Background
SC-GS was introduced as a variant of 3D Gaussian Splatting for dynamic scenes, focusing on sparse control for editing and compactness [S1, unofficial][S2, unofficial]. It builds on the canonical + deformation decomposition common in deformable 3DGS methods.

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
