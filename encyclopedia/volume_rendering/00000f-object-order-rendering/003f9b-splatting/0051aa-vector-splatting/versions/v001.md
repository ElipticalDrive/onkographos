# Vector Splatting

Also called: Combined Vector and Scalar Texture Splats, Texture Mapped Vector Splat, Vector Field Splatting

## Overview
Vector Splatting is a technique for visualizing vector fields by embedding direction indicators into scalar reconstruction texture maps, allowing the simultaneous rendering of scalar density and vector direction [S1, unofficial]. It transforms vectors to screen space and maps their magnitude to color, splatting them onto the image alongside the scalar field [S1, unofficial]. Unlike standard scalar splatting, it explicitly handles the orientation and foreshortening of vector indicators to provide directional cues [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D scalar field (density/intensity), a 3D vector field (direction/magnitude), a viewing matrix, and a two-component texture map containing scalar intensity and vector indicator patterns [S1, unofficial].
*   **Output:** A 2D image where scalar density is modulated by vector opacity and color [S1, unofficial].

**Data Structures:**
*   **Texture Map:** A two-component texture map is used [S1, unofficial].
    *   **Component 1 (Intensity Map):** Represents the scalar field intensity $I_s$ [S1, unofficial].
    *   **Component 2 (Opacity/Vector Map):** Represents the vector indicator opacity $I_v$ and direction [S1, unofficial].
    *   **Layout:** The texture is structured as a grid where columns represent foreshortening levels indexed by the viewing direction component $V_z$, and rows represent temporal phases for animation [S1, unofficial]. The source specifies a table of 16 maps [S1, unofficial].

**Steps:**
1.  **Pre-compute Inverse View Matrix:** Calculate the inverse of the viewing matrix once for the entire dataset (e.g., octree) to transform vectors from world space to screen/viewing coordinates [S1, unofficial].
2.  **Transform Vectors:** For each splat, determine the vector field direction and transform it to viewing coordinates using the pre-computed inverse matrix [S1, unofficial].
3.  **Calculate Rotation and Foreshortening:**
    *   Project the transformed vector $(v_x, v_y)$ to determine the rotation angle of the polygon splat [S1, unofficial].
    *   Determine the foreshortening based on the vector's component towards the eye ($V_z$) relative to the overall vector length [S1, unofficial].
    *   *Note:* Simply shortening the polygon in the vector direction can lead to noticeable thin splats or regions of no activity [S1, unofficial]. Changing texture mapping coordinates in the x-axis to simulate foreshortening cannot be automatically windowed by current hardware [S1, unofficial].
    *   *Solution:* Use a table of textures indexed by $V_z$ to select the appropriate foreshortening level [S1, unofficial].
4.  **Select Texture Coordinates:**
    *   Select the column in the texture map corresponding to the calculated foreshortening ($V_z$) [S1, unofficial].
    *   For animation, assign each splat a random index into the rows of the texture map. For animated flows, add a changing phase shift to this indexing over time [S1, unofficial].
5.  **Rasterize and Composite:**
    *   Render the polygon splat using the selected portion of the texture map [S1, unofficial].
    *   The polygon's base color $(R_c, G_c, B_c)$ and opacity $A_s$ are modified by the texture values $I_s$ and $I_v$ [S1, unofficial].
    *   The color components are computed as:
        $$ R = I_s \cdot (1 - I_v) + R_c \cdot I_v $$
        $$ G = I_s \cdot (1 - I_v) + G_c \cdot I_v $$
        $$ B = I_s \cdot (1 - I_v) + B_c \cdot I_v $$
    *   The final alpha is computed as:
        $$ A = A_s \cdot A_v $$
        (Where $A_v$ is derived from the vector opacity map, typically $I_v$) [S1, unofficial].
    *   The vector color $(R_c, G_c, B_c)$ can be modulated per splat to code for vector magnitude [S1, unofficial].
    *   If rendering the vector field alone (without the scalar background), use a modulate texture operation instead of a blend operation [S1, unofficial].

**Parameters:**
*   $I_s$: Intensity value from the scalar texture map [S1, unofficial].
*   $I_v$: Opacity/Vector indicator value from the vector texture map [S1, unofficial].
*   $R_c, G_c, B_c$: Color components of the vector, modifiable per splat for magnitude coding [S1, unofficial].
*   $A_s$: Polygon opacity (scalar field opacity) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Thin Splats:** Simply shortening the polygon in the vector direction leads to noticeable thin splats or regions of no activity [S1, unofficial].
*   **Hardware Limitations:** Changing texture mapping coordinates in the x-axis direction to simulate foreshortening cannot be automatically windowed by current hardware [S1, unofficial].
*   **Performance:** Vector splats incur approximately a 50% performance penalty due to the matrix multiply required for each vector [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The method incurs approximately a 50% performance penalty compared to scalar-only splats due to the matrix multiplication required to transform each vector to screen space [S1, unofficial]. The inverse viewing matrix is computed only once for the entire octree, but rotation and foreshortening calculations are performed per vector [S1, unofficial].
*   **Memory Complexity:** Requires storage for the two-component texture map (scalar intensity and vector opacity/direction) and the vector field data [S1, unofficial].
*   **Strengths:** Allows simultaneous visualization of scalar density and vector direction; provides better image quality than polygonal mesh approximations of Gaussian splats; supports animation of flow fields [S1, unofficial].
*   **Limitations:** Significant performance overhead (50% penalty); artifacts from simple polygon shortening; hardware limitations on dynamic texture coordinate windowing [S1, unofficial].
*   **When to use:** When both scalar density and vector direction/magnitude need to be visualized simultaneously in real-time or near-real-time, and the performance cost is acceptable [S1, unofficial].
*   **When not to use:** When maximum performance is critical and vector direction is not required; when hardware does not support the required texture operations [S1, unofficial].

## Background
Vector Splatting was developed as an extension to the VolumeToGeom and Render modules of the SGI Explorer system, using the Inventor graphics library [S1, unofficial]. It was designed to integrate vector fields into scalar reconstruction functions by adding disturbances such as tiny vector particles or scratch marks [S1, unofficial]. The technique was applied to visualize climate data, such as clouds (scalar) and winds (vector) over North America [S1, unofficial].

## Sources
[S1] Texture splats for 3D vector and scalar field visualization, https://digital.library.unt.edu/ark:/67531/metadc1339952/, fetched 2026-10-26 (unofficial)
