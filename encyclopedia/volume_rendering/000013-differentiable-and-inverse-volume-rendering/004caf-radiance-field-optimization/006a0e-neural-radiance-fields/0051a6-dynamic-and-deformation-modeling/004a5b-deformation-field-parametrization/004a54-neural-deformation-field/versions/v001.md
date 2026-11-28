# Neural Deformation Field

Also called: non-rigid deformation network

## Overview
The Neural Deformation Field (NDF) is a method for modeling non-rigid deformations in dynamic scenes by predicting a per-point displacement vector in canonical space, conditioned on pose and geometric attributes [S1, unofficial]. It solves the problem of representing articulated or soft-tissue motion that Linear Blend Skinning (LBS) cannot capture due to its rigid-bone assumption [S1, unofficial]. Unlike sibling methods that may bake geometry into static proxies, the NDF jointly optimizes the deformation network with the radiance field from scratch, allowing for direct geometry and texture editing [S1, unofficial].

## How it works
The method transforms a query point $\mathbf{x}_t$ at time $t$ (or pose $\theta_t$) to a canonical coordinate $\mathbf{x}_c$ by applying an inverse LBS transformation followed by a neural displacement [S1, unofficial]. The canonical point is then queried from a static radiance field $F_{\text{static}}$ to obtain density $\sigma$ and color $\mathbf{c}$ [S1, unofficial].

**Inputs:**
*   Target point in camera space $\mathbf{x}_t$.
*   Target pose parameters $\theta_t$.
*   Signed height indicator $\tilde{h}$, derived from the point's position relative to a reference mesh.

**Outputs:**
*   Canonical point $\mathbf{x}_c$.
*   Displacement vector $\Delta$.

**Data Structures:**
*   **Signed Height Indicator $\tilde{h}$**: A 3D vector $[u, v, \bar{d}]$ [S1, unofficial].
    *   $[u, v]$: 2D UV coordinates of the point $\mathbf{x}_t$ projected onto the reference mesh surface.
    *   $\bar{d}$: Signed distance from $\mathbf{x}_t$ to the mesh surface.
*   **Neural Network $F_\Delta$**: An MLP that takes position-encoded $\tilde{h}$ and pose $\theta_t$ as input and outputs $\Delta$.

**Step-by-Step Procedure:**

1.  **Project and Compute Signed Distance**:
    Project $\mathbf{x}_t$ onto the reference mesh surface at $\mathbf{x}_0$. Calculate the direction vector $\tilde{\mathbf{r}} = \mathbf{x}_t - \mathbf{x}_0$. Let $\tilde{\mathbf{n}}$ be the surface normal at $\mathbf{x}_0$. Compute the signed distance $\bar{d}$:
    $$
    \bar{d} = \text{sign}(\tilde{\mathbf{r}} \cdot \tilde{\mathbf{n}}) \|\tilde{\mathbf{r}}\|_2
    $$
    [S1, unofficial]

2.  **Construct Signed Height Indicator**:
    Form the vector $\tilde{h} = [u, v, \bar{d}]$ [S1, unofficial].

3.  **Encode and Predict Displacement**:
    Apply a position encoding function $\phi(\cdot)$ to $\tilde{h}$. Pass the encoded indicator and the target pose $\theta_t$ through the deformation network $F_\Delta$:
    $$
    \Delta = F_\Delta(\phi(\tilde{h}), \theta_t)
    $$
    [S1, unofficial]

4.  **Compute Canonical Point**:
    Apply the inverse LBS transformation to $\mathbf{x}_t$ and add the predicted displacement:
    $$
    \mathbf{x}_c = \text{LBS}^{-1}(\mathbf{x}_t) + \Delta
    $$
    [S1, unofficial]

5.  **Query Radiance Field**:
    Query the static radiance field $F_{\text{static}}$ at $\mathbf{x}_c$ to obtain density $\sigma$ and color $\mathbf{c}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Static Background**: If the distance between a query point and its nearest SMPL vertex exceeds a threshold, the point is classified as static background. In this case, bone weights are set to zero, background weight to one, and no deformation is applied [S1, unofficial].
*   **Precision**: The accuracy of $\bar{d}$ depends on the precision of the surface projection $\mathbf{x}_0$ and normal $\tilde{\mathbf{n}}$ [S1, unofficial].

## Subtopics

### Geometry Editing via Normal Compensation
This variant allows for the deformation of a neural implicit field by deforming a mesh scaffold while maintaining local consistency of learnable signed distances [S2]. It compensates for the rotation of surface normals during deformation by adjusting the signed indicator $\tilde{h}(x)$ [S2].
*   **Definition**: Deforming a neural implicit field by deforming the mesh scaffold while maintaining local consistency of learnable signed distances [S2].
*   **Formula**: The compensated signed indicator is calculated as:
    $$
    \tilde{h}'(x) = \tilde{h}(x) + \delta h_x
    $$
    where $\delta h_x$ is the relative difference of the vertex normal (averaged from nearby surface normals) from the original mesh to the deformed mesh [S2].
*   **When to choose**: Use when explicit geometry editing (e.g., swapping textures, deforming specific regions) is required on a mesh-based representation [S2].

## Operating characteristics
*   **Time Complexity**: Depends on the evaluation of $F_\Delta$ (MLP inference) and the position encoding $\phi(\cdot)$ per sample.
*   **Memory Complexity**: Requires storing the parameters of $F_\Delta$ and the static radiance field.
*   **Strengths**: Jointly optimizes geometry and deformation from scratch, avoiding manual parameter tuning required by two-stage distilling methods [S1, unofficial].
*   **Limitations**: Relies on the accuracy of the reference mesh for UV mapping and signed distance calculation [S1, unofficial].

## Background
The Neural Deformation Field was introduced in the context of unified volumetric avatars for view synthesis, pose rendering, and geometry editing [S1, unofficial]. It addresses the limitations of rigid LBS by introducing a neural network to predict non-rigid displacements conditioned on pose and geometric attributes [S1, unofficial].

## Sources
[S1] UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing, https://doi.org/10.48550/arXiv.2304.06969, fetched 2026-10-29 (unofficial)
[S2] NeuMesh: Learning Disentangled Neural Mesh-Based Implicit Field for Geometry and Texture Editing, https://doi.org/10.1007/978-3-031-19787-1_34, fetched 2026-10-26
