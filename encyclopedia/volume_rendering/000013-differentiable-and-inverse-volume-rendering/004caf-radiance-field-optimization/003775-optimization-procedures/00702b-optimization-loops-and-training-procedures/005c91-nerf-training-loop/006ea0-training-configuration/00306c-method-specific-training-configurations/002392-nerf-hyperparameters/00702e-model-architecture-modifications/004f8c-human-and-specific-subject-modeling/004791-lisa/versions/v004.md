# LISA

Also called: LISA (April 2022)

## Overview
LISA is a part-based implicit neural representation method specifically designed for modeling human hands [S1]. It approximates the hand structure as a collection of rigid parts, leveraging the MANO (Modeling of the HAND) parametric model to define the underlying skeletal and shape topology [S2]. Unlike monolithic NeRF approaches, LISA decouples geometry and appearance by using Signed Distance Fields (SDFs) for shape description and a separate color field for realistic rendering [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** 3D query points $\mathbf{q}$, MANO skeletal posture parameters (joint angles), and MANO shape parameters (implicitly derived from posture).
*   **Outputs:** Signed Distance Field value $d(\mathbf{q})$ representing geometry, and color $\mathbf{c}(\mathbf{q})$ representing appearance.

**Data Structures:**
*   **MANO Model:** A parametric hand model providing a skeletal hierarchy and mesh topology. The shape parameters $\beta$ are not learned directly as independent variables but are implicitly described by the current skeletal posture [S2].
*   **MLPs:** Two distinct Multi-Layer Perceptrons (or a shared backbone with separate heads) are employed. One network predicts the SDF value, and the other predicts the RGB color [S1].

**Steps:**
1.  **Pose-Driven Shape Definition:** Given a specific hand pose, the MANO model defines the spatial configuration of the hand's rigid parts (bones and fingers) [S2]. The shape parameters $\beta$ are implicitly determined by this skeletal posture, ensuring the geometry adheres to anatomical constraints [S2].
2.  **Query Processing:** For each 3D query point $\mathbf{q}$ in the scene:
    a. Determine the local coordinate system or feature representation relative to the nearest rigid part or bone defined by the MANO skeleton [S1].
    b. Input $\mathbf{q}$ and relevant pose-dependent features into the geometry MLP.
    c. Input $\mathbf{q}$ and relevant pose-dependent features into the color MLP.
3.  **Geometry Prediction:** The geometry MLP outputs the Signed Distance Field value $d(\mathbf{q})$, which represents the distance from $\mathbf{q}$ to the hand surface [S2]. This SDF representation allows for precise boundary definition and avoids the "fuzzy" density fields of standard NeRFs [S2].
4.  **Color Prediction:** The color MLP outputs the view-dependent color $\mathbf{c}(\mathbf{q})$ [S1].
5.  **Rendering:** Standard volume rendering or ray marching is performed using the SDF to determine surface intersections and the color field to shade the surface [S1]. Physical constraints may be introduced to prevent hand-object penetration if interacting objects are present [S2].

**Formulas:**
*   Geometry: $d = \text{MLP}_{geo}(\mathbf{q}, \text{pose})$
*   Color: $\mathbf{c} = \text{MLP}_{color}(\mathbf{q}, \text{pose})$
*   Shape Constraint: $\beta = f(\text{pose})$ (Implicitly described by skeletal posture) [S2].

**Parameters:**
*   MANO model parameters (joint angles, shape coefficients) [S2].
*   MLP hyperparameters (layers, hidden dimensions) [S1].

**Edge Cases and Pitfalls:**
*   **Penetration:** Without explicit physical constraints, the hand mesh may intersect with other objects. LISA addresses this by introducing physical constraints during optimization [S2].
*   **Part Boundaries:** Since the hand is modeled as rigid parts, discontinuities may occur at joint boundaries if not handled smoothly by the MLPs [S1].

## Subtopics

### Unified Volumetric Avatar (UVA)
Focuses on full-body avatars with unified volumetric representation, whereas LISA is specialized for hands using part-based SDFs [Link](13784-unified-volumetric-avatar-uva.md).

### HumanNeRF
Uses a standard NeRF architecture for full-body humans, lacking the explicit part-based SDF structure of LISA [Link](13842-humannerf.md).

### HumanNeRF-SE
Extends HumanNeRF with semantic embeddings, but does not use the MANO-based part decomposition [Link](15640-humannerf-se.md).

### Dynamic Semantic Human NeRF
Incorporates semantic labels and dynamics for full-body, whereas LISA is static/part-based for hands [Link](21763-dynamic-semantic-human-nerf.md).

## Operating characteristics
*   **Time Complexity:** Depends on the number of query points and MLP depth. SDF evaluation is generally faster than density integration for surface extraction [S1].
*   **Memory Complexity:** Lower than dense voxel grids due to implicit representation, but requires storing MANO model data [S2].
*   **Strengths:** High-fidelity hand geometry via SDF; realistic appearance via learned color field; anatomical plausibility via MANO [S2].
*   **Limitations:** Limited to hand modeling; requires accurate pose estimation for MANO parameters [S1].
*   **When to use:** When high-quality, animatable hand avatars are needed from image sequences [S1].
*   **When not to use:** For full-body avatars or non-hand objects [S1].

## Background
LISA was introduced in April 2022 as a method specifically targeting the modeling of human hands [S1]. It emerged from the broader research area of neural radiance fields for 3D avatars, distinguishing itself by approximating hands as a collection of rigid parts rather than a continuous volume [S1]. It builds upon the MANO parametric hand model, using it to implicitly describe shape parameters via skeletal posture [S2]. This approach addresses the complexity of hand topology and the need for realistic appearance in hand avatars [S2].

## Sources
[S1] Neural Radiance Fields in 3D Vision: A Comprehensive Review, https://doi.org/10.26599/cvm.2026.9450535, fetched 2026-10-28
[S2] Human 3D Avatar Modeling with Implicit Neural Representation: A Brief Survey, https://doi.org/10.1109/icsps58776.2022.00148, fetched 2026-10-26
