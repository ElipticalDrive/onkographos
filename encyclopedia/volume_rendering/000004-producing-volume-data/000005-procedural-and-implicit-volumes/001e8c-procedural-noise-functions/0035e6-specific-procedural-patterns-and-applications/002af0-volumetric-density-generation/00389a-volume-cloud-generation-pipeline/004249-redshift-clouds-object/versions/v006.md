# Redshift Clouds Object
Also called: Redshift Clouds
## Overview
The Redshift Clouds Object is a procedural volume primitive that generates entire procedural skies [S1]. It distinguishes itself from other volumetric methods by offering two distinct rendering modes: a "fast mode" that renders the cloud deck as a clever background, and a "fully volumetric mode" that allows geometry such as skyscrapers or aircraft to pierce the cloud layer and be lit by it [S1]. Unlike the Redshift Environment Object, which fills the scene with a homogeneous scattering medium for atmospheric haze and god rays, the Clouds Object is designed specifically for sky generation [S1].
## How it works
The method relies on procedural generation rather than voxel data [S1].

**Inputs and Outputs:**
*   **Input:** Scene parameters defining the sky volume; camera position and orientation.
*   **Output:** Rendered image with volumetric sky effects.

**Data Structures:**
Not given in the sources. The implementation details of the procedural noise functions or density fields are not specified [S1].

**Steps:**
1.  **Mode Selection:** The user selects between "fast mode" and "fully volumetric mode" [S1].
2.  **Fast Mode Execution:** If selected, the cloud deck is rendered as a background element [S1]. This mode likely avoids full volumetric ray marching for the sky volume to optimize performance [S1].
3.  **Volumetric Mode Execution:** If selected, the cloud layer is treated as a full volume [S1].
    *   Ray marching is performed through the procedural sky volume [S1].
    *   Geometry (e.g., buildings, aircraft) is allowed to intersect the volume [S1].
    *   Lighting calculations account for the interaction between the geometry and the volumetric cloud layer [S1].

**Parameters:**
Not given in the sources. Specific parameters for controlling cloud density, color, or procedural noise are not detailed in the provided text [S1].

**Edge Cases and Pitfalls:**
Not given in the sources.

## Subtopics
## Operating characteristics
*   **Performance:** The "fast mode" is optimized for speed by rendering the cloud deck as a background [S1]. The "fully volumetric mode" is computationally more expensive as it requires full volumetric calculations [S1].
*   **Use Cases:** Ideal for simulating procedural skies where camera or geometry may penetrate the cloud layer [S1].
*   **Limitations:** Specific performance metrics or resolution limits are not provided [S1].

## Background
The Redshift Clouds Object is part of the Redshift rendering engine's volumetric tools [S1]. It is designed to work alongside other Redshift volumetric primitives like the Environment Object [S1].

## Sources
[S1] Introduction to Volumetric Rendering, https://help.maxon.net/c4d/2027/en-us/Content/html/What_is_Volumetric_Rendering.html, fetched 2026-10-29
