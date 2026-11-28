# Maximum-Value Projection Shading Handling

Also called: Maximum Shader

## Overview
Maximum-Value Projection Shading Handling is a method for rendering volumetric data where the final pixel color is determined by the highest scalar value encountered along the viewing ray, rather than an accumulation or average. It solves the problem of visualizing the outermost or most significant features in a volume while suppressing internal structures. It differs from [Exponential Fog (Beer-Lambert)](3968-exponential-fog-beer-lambert.md) and [Absorption Adjustment](4020-absorption-adjustment.md) by discarding all intermediate data values and focusing solely on the extremum, and from [Non-Exponential Free Flight Model](4593-non-exponential-free-flight-model.md) by not modeling probabilistic light transport but rather deterministic data extraction.

## How it works
The method combines a data extraction shader with specific shading adjustments to ensure correct visibility of underlying geometry through the volume.

**Inputs and Outputs:**
*   **Input:** A ray cast through a volumetric cell containing scalar data values.
*   **Output:** A single color value derived from the maximum scalar value found along the ray, processed through a transfer function.

**Data Structures:**
*   **Ray:** Defined by origin $O$ and direction $D$.
*   **Cell:** A volumetric element with an associated opacity $O_1$ and scalar data values.
*   **Transfer Function:** A mapping $T(v) \rightarrow C$ that converts a scalar value $v$ to a color $C$.

**Steps:**
1.  **Ray Casting:** Cast a primary ray through the volume.
2.  **Data Sampling:** Examine all measured data values along the ray path within the current cell [S2].
3.  **Maximum Extraction:** Identify the maximum scalar value $v_{max}$ from the sampled data [S2].
4.  **Color Mapping:** Apply the transfer function to $v_{max}$ to determine the base color [S2].
5.  **Shading Configuration:**
    *   Turn off all shading effects except transmission [S1, unofficial].
    *   Set the index of refraction to 1 [S1, unofficial].
    *   Calculate the transmission coefficient $T_c$ using the opacity $O_1$ of the maximum value:
        $$ T_c = 1 - O_1 $$
        where $O_1$ is the opacity corresponding to the maximum value [S1, unofficial].
6.  **Intersection Handling (Recursive Prevention):**
    *   If a transmission ray is cast from a maximum-value intersection point, the current cell must be turned "off" to prevent the ray from performing intersection calculations with it, thereby avoiding a recursive maximum-value intersection within the same object [S1, unofficial].
    *   Once the transmission ray intersects the target polygon, the cell must be turned "on" again to allow correct shadowing and reflection calculations for subsequent rays [S1, unofficial].

**Parameters:**
*   **Index of Refraction:** Set to 1 [S1, unofficial].
*   **Transmission Coefficient:** $1 - O_1$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Recursive Intersection:** Naively casting a transmission ray from a maximum-value intersection without disabling the current cell results in another maximum-value intersection within the same object, causing visual artifacts or infinite loops [S1, unofficial].
*   **Opacity Handling:** The transmission coefficient depends on the opacity of the maximum value. If the maximum value is opaque ($O_1 = 1$), no transmission occurs, and underlying polygons are hidden [S1, unofficial].

## Subtopics

### Maximum Shader
A shader that examines all measured data values along the ray and displays the maximum found using a transfer function [S2]. It is chosen when the user wants to visualize the most significant or outermost scalar values in a volume, such as the highest density or temperature, ignoring lower values. It is distinct from the Accumulation shader, which sums values, and the Cloud shader, which adds lighting functions [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples taken along the ray to find the maximum.
*   **Memory Complexity:** Minimal, requiring storage for the current maximum value and the transfer function.
*   **Strengths:** Provides clear visualization of dominant features; computationally efficient for data extraction.
*   **Limitations:** Loses information about internal structures; sensitive to the transfer function definition.
*   **When to use:** When the goal is to highlight the most extreme values in a dataset.
*   **When not to use:** When internal details or average properties are of interest.

## Background
The concept of the "Maximum shader" was introduced in the context of volume rendering in architecture, specifically for overlapping and combining 3D voxel volume data with 3D building models [S2]. The shading handling details, including the transmission coefficient formula and intersection management, were described in the context of volumetric ray tracing within the VolVis volume visualization system [S1, unofficial].

## Sources
[S1] Volumetric Ray Tracing, https://www.cs.drexel.edu/~david/Classes/Papers/p11-sobierajski.pdf, fetched 2026-10-28 (unofficial)
[S2] Volume Rendering in Architecture: Overlapping and combining 3d voxel volume data with 3d building models, https://doi.org/10.52842/conf.ecaade.2012.1.719, fetched 2026-10-26
