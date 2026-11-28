# Object Manipulation

Also called: Face manipulation

## Overview
Object manipulation allows users to directly alter the geometry or orientation of 3D data, such as changing the curvature of a model, to support inter-user understanding in collaborative settings [S1]. It is distinguished from [Ray-based Picking](19665-ray-based-picking.md) by its focus on modifying the scene state (e.g., dragging faces) rather than merely selecting data for visualization, and from [Tangible Dataset Manipulation](#tangible-dataset-manipulation) by its reliance on standard input devices and depth rays rather than physical reference objects.

## How it works
**Inputs and Outputs:**
*   **Input:** User input device movements (e.g., mouse, stylus); depth ray generated from the view.
*   **Output:** Modified 3D model geometry (e.g., displaced faces) or rotated volume dataset.

**Data Structures:**
*   **Model Representation:** A mesh or volume representation where individual faces or voxels are addressable.
*   **Selection State:** A set of selected faces identified by the depth ray.
*   **Transformation State:** For face manipulation, a displacement vector along the face normal; for rotation, a 6-DOF pose.

**Steps:**
1.  **Selection:** Cast a depth ray from the user's viewpoint through the input device position to intersect the 3D model [S1]. Identify the faces intersected by this ray [S1].
2.  **Activation:** The user initiates a drag operation on the selected faces [S1].
3.  **Transformation (Face):** For each selected face, calculate its normal vector. Move the face along its normal vector proportional to the input device movement [S1].
4.  **Transformation (Tangible):** Map the rotation of a physical reference object to the rotation of the virtual volume dataset [S2].
5.  **Rendering:** Render the modified scene or rotated volume [S1, S2].

**Formulas:**
*   Face Displacement: $P_{new} = P_{old} + \alpha \cdot \vec{n} \cdot d$
    *   $P$: Face position.
    *   $\vec{n}$: Face normal vector.
    *   $d$: Input device displacement magnitude.
    *   $\alpha$: Scaling factor (Not given in the sources.).

**Parameters:**
*   **Depth Ray:** Used for selection [S1].
*   **Reference Object Shape:** Cuboctahedron (for tangible manipulation) to ensure at least one marker remains visible during rotation [S2].

**Edge Cases and Pitfalls:**
*   **Occlusion:** In tangible manipulation, fingers may obscure markers on the reference object, hiding the last visible marker [S2].
*   **Parallel Access:** Users can modify the scene in parallel, but if areas are too far apart or require different zoom levels, parallel access may fail without scene splitting [S1].

## Subtopics

### Tangible Dataset Manipulation
A technique where rotating a physical reference object (shaped as a cuboctahedron) rotates the corresponding virtual volume dataset, providing natural 6 DOF interaction [S2]. It is chosen when natural, bare-hand exploration of volume orientation is preferred over precise digital control. It differs from standard object manipulation by using physical markers and a handheld window/tablet for tracking rather than a depth ray [S2].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Supports inter-user understanding [S1]; allows natural 6 DOF interaction with bare hands [S2]; supports parallel access [S1].
*   **Limitations:** Requires specific hardware (depth ray system or physical reference object with markers); scene splitting is needed for distant areas or different zoom levels [S1]; finger occlusion can hinder tangible manipulation [S2].
*   **When to use:** Collaborative design reviews where demonstrating geometric changes (e.g., curvature) is critical [S1]; natural exploration of volume orientation [S2].
*   **When not to use:** When precise, fine-grained control without physical artifacts is required; when users are too far apart to share the same physical reference object.

## Background
Originated in collaborative volumetric display research. [S1] introduced face manipulation for car designers to demonstrate curvature changes. [S2] introduced tangible manipulation using a cuboctahedron reference object for portable volume exploration.

## Sources
[S1] Collaborative interaction with volumetric displays, https://doi.org/10.1145/1357054.1357118, fetched 2026-10-28
[S2] A portable interface for tangible exploration of volumetric data, https://doi.org/10.1145/2671015.2671130, fetched 2026-10-29
