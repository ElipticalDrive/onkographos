# Neural Deformation Field

Also called: non-rigid deformation network

## Overview
The Neural Deformation Field (NDF) is a method for modeling non-rigid, time-varying deformations in neural radiance fields by predicting per-point displacements in a canonical space [S1, unofficial]. It solves the problem of animating static neural representations without relying on rigid body assumptions or human-specific priors like SMPL [S1, unofficial]. It sets itself apart from sibling methods like Deformation-Guided Neural Fields by explicitly decoupling the rigid skeletal transformation (LBS) from the residual non-rigid displacement predicted by a neural network [S1, unofficial].

## How it works
The method operates by mapping points from a target deformed space to a canonical space using a combination of inverse Linear Blend Skinning (LBS) and a neural displacement field [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A point in the target deformed space $\mathbf{x}_t$, the target pose $\theta_t$, and the 2D UV coordinates $[u, v]$ of the point on the underlying mesh surface [S1, unofficial].
*   **Output:** The corresponding point $\mathbf{x}_c$ in the canonical space [S1, unofficial].

**Data Structures:**
*   **Signed Height Indicator ($\tilde{h}$):** A vector encoding the geometric relationship of the point to the mesh surface. It is defined as $\tilde{h} = [u, v, \bar{d}]$, where $[u, v]$ are the 2D UV coordinates and $\bar{d}$ is the signed distance [S1, unofficial].
*   **Neural Network ($F_\Delta$):** An MLP that takes the position-encoded signed height indicator and the target pose as input to predict the displacement $\Delta$ [S1, unofficial].

**Step-by-Step Method:**

1.  **Project to Mesh Surface:** Given a target point $\mathbf{x}_t$, project it onto the mesh surface to find the nearest surface point $\mathbf{x}_0$ [S1, unofficial].
2.  **Calculate Signed Distance ($\bar{d}$):**
    *   Compute the direction vector from the surface point to the target point: $\tilde{\mathbf{r}} = \mathbf{x}_t - \mathbf{x}_0$ [S1, unofficial].
    *   Obtain the surface normal $\tilde{\mathbf{n}}$ at $\mathbf{x}_0$ [S1, unofficial].
    *   Calculate the signed distance:
        $$
        \bar{d} = \text{sign}(\tilde{\mathbf{r}} \cdot \tilde{\mathbf{n}}) \|\tilde{\mathbf{r}}\|_2
        $$
        [S1, unofficial]
3.  **Construct Signed Height Indicator:** Combine the UV coordinates and signed distance:
    $$
    \tilde{h} = [u, v, \bar{d}]
    $$
    [S1, unofficial]
4.  **Predict Displacement ($\Delta$):** Pass the position-encoded signed height indicator $\phi(\tilde{h})$ and the target pose $\theta_t$ through the neural deformation network $F_\Delta$:
    $$
    \Delta = F_\Delta(\phi(\tilde{h}), \theta_t)
    $$
    [S1, unofficial]
5.  **Compute Canonical Point ($\mathbf{x}_c$):** Apply the inverse LBS transformation to the target point and add the predicted displacement:
    $$
    \mathbf{x}_c = LBS^{-1}(\mathbf{x}_t) + \Delta
    $$
    [S1, unofficial]

**Parameters:**
*   $\phi(\cdot)$: Position encoding function [S1, unofficial].
*   $LBS^{-1}$: Inverse Linear Blend Skinning transformation [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Static Background:** If a point's nearest SMPL vertex exceeds a certain threshold, it is classified as a static background point. In this case, the bone weight is set to zero, the background weight to one, and no non-rigid deformation is applied [S1, unofficial].
*   **Precision:** The calculation of $\bar{d}$ relies on the accuracy of the surface normal $\tilde{\mathbf{n}}$ and the projection $\mathbf{x}_0$. Errors in mesh projection can lead to incorrect signed distances, affecting the displacement prediction [S1, unofficial].

## Subtopics

### [Dynamic Point Field Deformation](14823-dynamic-point-field-deformation.md)
Dynamic Point Field Deformation (DPF) is a variant that models non-rigid, time-varying deformations in point-based radiance fields by predicting the translation between a canonical space point cloud and a deformed space point cloud using a neural network. It solves the problem of animating static neural point clouds without relying on rigid body assumptions or human-specific priors like SMPL. Unlike cage-based or linear blend skinning methods, DPF generalizes to arbitrary objects and enables fine-level complex deformation through user-defined or algorithmically extracted key points. See [Dynamic Point Field Deformation](14823-dynamic-point-field-deformation.md).

### Geometry Editing via Normal Compensation
This variant addresses the consistency of learnable signed distances when deforming a neural implicit field by deforming the mesh scaffold. To maintain local consistency, the rotation of the surface normal is compensated in the learnable signed indicator $\tilde{h}(x)$ [S2]. The compensated signed indicator is calculated as:
$$
\tilde{h}'(x) = \tilde{h}(x) + \delta h_x
$$
where $\delta h_x$ is the relative difference of the vertex normal (averaged from nearby surface normals) from the original mesh to the deformed mesh [S2]. This ensures that the interpolated signed distance of locally deformed or rotated regions remains consistent [S2].

## Operating characteristics
Not given in the sources.

## Background
The Neural Deformation Field was introduced in the context of unified volumetric avatars for view synthesis, pose rendering, and geometry editing [S1, unofficial]. It was proposed to address the limitations of previous methods that relied on baking polygon meshes from pre-trained neural fields, which required manual parameter tuning [S1, unofficial]. The method jointly optimizes the geometry field with other components from scratch [S1, unofficial].

## Sources
[S1] UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing, https://doi.org/10.48550/arXiv.2304.06969, fetched 2026-10-29 (unofficial)
[S2] NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing, https://doi.org/10.1007/978-3-031-19787-1_34, fetched 2026-10-26
