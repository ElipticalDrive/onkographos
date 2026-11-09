# Deep Shadow Maps

Also called: DOM, Deep Opacity Maps

## Overview
Deep Shadow Maps are a shadow mapping technique for volume rendering where each pixel stores a curve representing volume density as a function of distance, rather than just the depth to the nearest opaque object [S1, unofficial]. This allows for accurate occlusion calculations in participating media by integrating transmission along the ray [S1, unofficial]. Unlike standard shadow maps which only capture the nearest surface, deep shadow maps capture the entire volumetric structure along the light ray [S1, unofficial].

## How it works
**Inputs:**
*   Scene geometry and volume density fields.
*   Light positions and directions.
*   Camera parameters.

**Outputs:**
*   A set of shadow maps, one per light source.
*   Each shadow map pixel contains a data structure representing the volume density profile along the ray from the light to the far plane.

**Data Structures:**
Each pixel in the deep shadow map stores a curve. While the exact bit-packing is not specified in the primary sources, the conceptual structure is a function $D(d)$ where $d$ is the distance from the light source and $D$ is the density or transmission factor [S1, unofficial]. In practice, this is often implemented as a 1D texture or a series of samples along the ray [S1, unofficial].

**Steps:**
1.  **Light Space Projection:** For each light in the scene, project the volume geometry into the light's coordinate space [S1, unofficial].
2.  **Volume Rasterization:** Render the volume into the shadow map. Instead of storing a single depth value, store the density information as a function of distance [S1, unofficial].
3.  **Curve Storage:** Each pixel in the shadow map stores a curve representing how transmission through the volume changes as one moves through it [S1, unofficial]. This curve effectively describes the optical depth along the ray.
4.  **Shadow Lookup:** During volume rendering, for each sample point on the view ray, project its position into the light's shadow map [S1, unofficial].
5.  **Transmission Calculation:** Retrieve the density/transmission value from the deep shadow map at the projected location. Integrate this value along the ray to compute the final shadowed illumination [S1, unofficial].

**Formulas:**
The transmission $T$ along a ray segment from $d_1$ to $d_2$ is computed by integrating the density $\sigma$ stored in the deep shadow map:
$$ T = \exp\left(-\int_{d_1}^{d_2} \sigma(d) \, dd\right) $$
Where $\sigma(d)$ is the density value retrieved from the deep shadow map at distance $d$ [S1, unofficial].

**Parameters:**
*   Shadow map resolution: Not given in the sources.
*   Number of samples per ray for curve approximation: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Storing a continuous curve in discrete pixels requires careful sampling and interpolation to avoid artifacts [S1, unofficial].
*   **Memory:** Deep shadow maps consume significantly more memory than standard shadow maps due to the storage of multiple values per pixel [S1, unofficial].
*   **Complexity:** The integration step during rendering adds computational cost compared to simple depth comparisons [S1, unofficial].

## Subtopics

### Cascaded Deep Opacity Maps
A variant of deep opacity maps using a series of cuboids covering different splits of the view frustum, similar to cascaded shadow maps, to handle varying distances [S2, unofficial]. This approach improves accuracy for volumes at different depths by allocating higher resolution to closer regions [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Higher than standard shadow maps due to the need to store and integrate density curves [S1, unofficial].
*   **Memory Complexity:** Significantly higher than standard shadow maps as each pixel stores a curve rather than a single depth value [S1, unofficial].
*   **Strengths:** Accurate shadowing for participating media; handles complex volumetric structures [S1, unofficial].
*   **Limitations:** High memory usage; complex implementation; primarily of historical interest [S1, unofficial].
*   **When to use:** When accurate volumetric shadows are required and memory is not a constraint [S1, unofficial].
*   **When not to use:** For real-time applications with strict memory budgets or simple opaque scenes [S1, unofficial].

## Background
Deep Shadow Maps were developed in 2000 by Tom Lokovic and Eric Veach from Pixar [S1, unofficial]. The technique was introduced as a way to handle shadows in volumetric scenes where standard shadow maps fail due to their inability to represent partial occlusion within a volume [S1, unofficial]. It is often mentioned for completeness and historical reasons, with similarities to baking lighting into voxels [S1, unofficial].

## Sources
[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
