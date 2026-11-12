# Dynamic 3D Gaussian Splatting

Also called: 3DGS with temporal constraints, Dynamic 3DGS

## Overview
Dynamic 3D Gaussian Splatting enables real-time novel view synthesis of dynamic scenes by maintaining a persistent set of Gaussian primitives whose parameters evolve over time [S1, unofficial]. Unlike per-frame baselines that suffer from storage explosion and temporal flicker, this method tracks the position and rotation of each Gaussian across frames using local-rigid and As-Rigid-As-Possible (ARAP) or isometry constraints [S1, unofficial]. It distinguishes itself from deformation-field approaches (e.g., [4D Gaussian Splatting](7992-4d-gaussian-splatting.md)) by avoiding global deformation networks, instead relying on explicit temporal constraints on the primitive parameters themselves.

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of multi-view images $\{I_t\}_{t=1}^T$ capturing a dynamic scene.
*   **Outputs:** A set of $N$ Gaussians, each defined by a canonical position $\boldsymbol{\mu}_0$, scale $\mathbf{s}_0$, rotation $q_0$, opacity $\alpha_0$, and spherical harmonic coefficients $\mathbf{c}_0$. At any time $t$, the rendered image is produced by evaluating the time-varying position $\boldsymbol{\mu}_t$ and rotation $q_t$.

**Data Structures:**
Each Gaussian $i$ stores:
*   $\boldsymbol{\mu}_0 \in \mathbb{R}^3$: Canonical 3D position.
*   $\mathbf{s}_0 \in \mathbb{R}^3$: Canonical scale (log-space).
*   $q_0 \in \mathbb{R}^4$: Canonical rotation (quaternion, normalized).
*   $\alpha_0 \in \mathbb{R}$: Opacity.
*   $\mathbf{c}_0$: Spherical harmonic coefficients for view-dependent color.
*   *Note:* The method does not store explicit time-varying parameters for all frames; $\boldsymbol{\mu}_t$ and $q_t$ are computed on-the-fly or updated during optimization based on constraints.

**Optimization and Constraints:**
The core mechanism is the tracking of time-varying position $\boldsymbol{\mu}_t$ and rotation $q_t$ for each Gaussian. The optimization minimizes a photometric loss $\mathcal{L}_{img}$ between rendered and ground-truth images, augmented by temporal regularization terms.

1.  **Initialization:** Initialize Gaussians from the first frame using standard 3DGS initialization [Parent](1795-3d-gaussian-splatting.md).
2.  **Temporal Tracking:** For each frame $t$, the position $\boldsymbol{\mu}_t$ and rotation $q_t$ are derived from their canonical values $\boldsymbol{\mu}_0, q_0$ and a set of learned or optimized deformation parameters.
3.  **Local-Rigid Constraint:** To preserve the local geometry of the scene, the deformation between frames is constrained to be locally rigid. This prevents shearing and unnatural stretching of the Gaussian primitives.
4.  **ARAP / Isometry Constraint:** An As-Rigid-As-Possible (ARAP) or isometry constraint is applied to further regularize the deformation. This ensures that the transformation of the Gaussian cloud between time steps $t$ and $t+1$ minimizes the deviation from a rigid body transformation, preserving the intrinsic geometry of the dynamic object.
    *   Let $\mathbf{X}_t$ be the matrix of positions of all Gaussians at time $t$. The constraint minimizes:
        $$ \mathcal{L}_{temp} = \sum_{t} \| \mathbf{X}_{t+1} - (\mathbf{R}_t \mathbf{X}_t + \mathbf{t}_t) \|_F^2 $$
        where $\mathbf{R}_t$ and $\mathbf{t}_t$ are the optimal rotation and translation aligning $\mathbf{X}_t$ to $\mathbf{X}_{t+1}$ [S1, unofficial].
5.  **Rendering:** At each training step for frame $t$, the Gaussians are transformed to $\boldsymbol{\mu}_t, q_t$ and rendered using the standard 3DGS rasterizer.

**Parameters:**
*   Learning rate for Gaussian parameters: Standard 3DGS values.
*   Temporal loss weight: Not given in the sources.

**Edge Cases:**
*   **Occlusion:** If a Gaussian is occluded in a frame, its tracking may drift. The local-rigid constraint helps mitigate this by relying on neighboring visible Gaussians.
*   **Rapid Motion:** Large displacements between frames may violate the local-rigid assumption if the deformation is too large. [G] In practice, this may require adaptive learning rates or densification strategies to capture fast-moving details.

## Subtopics

### SpeeDe3DGS
SpeeDe3DGS accelerates deformable 3DGS by applying temporal pruning [S2, unofficial]. It identifies static and active Gaussians and groups the dynamic parts by motion similarity to reduce the number of deformation queries [S2, unofficial]. This optimization reduces computational overhead without significantly compromising rendering quality.

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ per frame, where $N$ is the number of Gaussians. Optimization time is higher than static 3DGS due to the additional temporal constraints and tracking calculations.
*   **Memory Complexity:** $O(N)$, storing only canonical parameters and potentially small deformation fields or constraints. This is significantly more compact than per-frame baselines.
*   **Performance:** Enables real-time rendering of dynamic scenes. The specific frame rates depend on the hardware and scene complexity, but the method is designed for real-time applicability.
*   **Strengths:** Persistent representation avoids storage explosion; temporal coherence reduces flicker; local-rigid constraints preserve geometry.
*   **Limitations:** May struggle with highly non-rigid deformations that violate the ARAP/isometry assumptions; requires careful tuning of temporal loss weights.

## Background
Dynamic 3D Gaussian Splatting was introduced by Luiten et al. in 3DV'24 [S1, unofficial]. It addresses the limitations of per-frame 3DGS reconstruction, which suffers from high storage costs and temporal inconsistencies. By extending static 3DGS with temporal tracking and deformation constraints, it enables persistent dynamic view synthesis.

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
[S2] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
