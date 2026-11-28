# Per-Fragment Phong Lighting with Gradient Normals

Also called: Gradient-based Phong Shading, Phong Lighting with Gradient Normals, Phong Shading with Gradient Normals

## Overview
This method applies per-fragment Phong or Gouraud shading to volume rendering by estimating surface normals from the gradient of the scalar field [S1, S2]. It solves the problem of visual flatness in unlit volume data by providing a "realistic" 3D impression through directional lighting [S2]. Unlike its sibling [Phong Shading with Boundary Enhancement and Facing Subtraction](15408-phong-shading-with-boundary-enhancement-and-facing-subtracti.md), which modulates alpha based on gradient magnitude and view direction for silhouette enhancement, this method relies strictly on the gradient for the normal vector and a standard dot-product diffuse/specular model [S1, S2]. It builds on [SDF Phong Lighting](4260-sdf-phong-lighting.md) by adapting the lighting model to scalar field gradients rather than explicit SDF normals.

## How it works
**Inputs:**
*   Scalar field $V$ (density values).
*   Threshold value $T$ (selects density region for shading).
*   Light vector $\mathbf{l}$ (moves with the view vector).
*   View vector $\mathbf{v}$ (implicit in the light vector movement).
*   Interpolation mode: Nearest neighbor or trilinear [S2].
*   Highlight flag: Boolean to enable specular highlights [S2].

**Outputs:**
*   Shaded color for each fragment/voxel where density $> T$ [S2].

**Data Structures:**
*   Scalar field $V$: Array of density values. Range 0–4095 [S2].
*   Gradient $\nabla V$: Estimated via finite differences of $V$ [S2].

**Steps:**
1.  **Gradient Calculation:** For each fragment, calculate the normal vector $\mathbf{n}$ as the gradient of the scalar field at that position. The gradient is computed using the selected interpolation mode (nearest neighbor or trilinear) to sample neighboring density values [S2].
    $$ \mathbf{n} = \nabla V(x, y, z) $$
    [G] The gradient is typically normalized to unit length: $\mathbf{n} \leftarrow \mathbf{n} / \|\mathbf{n}\|$. If $\|\mathbf{n}\| \approx 0$, the normal is undefined; a default normal (e.g., $(0,0,1)$) or no shading may be applied [G].

2.  **Threshold Check:** Determine if the current density value $V(x,y,z)$ exceeds the threshold $T$. Shading is applied only if $V(x,y,z) > T$ [S2].

3.  **Lighting Calculation:** Calculate the shading color using the Phong or Gouraud model [S2].
    *   **Diffuse Component:** Calculate the dot product between the normal $\mathbf{n}$ and the light vector $\mathbf{l}$.
        $$ C_{diffuse} = \max(0, \mathbf{n} \cdot \mathbf{l}) $$
    *   **Specular Component (Optional):** If highlights are enabled, calculate the specular reflection based on the view vector $\mathbf{v}$, light vector $\mathbf{l}$, and normal $\mathbf{n}$ using the Phong reflection model:
        $$ \mathbf{r} = 2(\mathbf{n} \cdot \mathbf{l})\mathbf{n} - \mathbf{l} $$
        $$ C_{specular} = \max(0, \mathbf{r} \cdot \mathbf{v})^{\alpha} $$
        where $\alpha$ is the shininess exponent (Not given in the sources).
    *   **Total Color:** Combine ambient, diffuse, and specular components. The base color is derived from the scalar field value [S2].

4.  **Interpolation:** Use nearest neighbor or trilinear interpolation to sample density values for gradient calculation and shading color determination [S2].

**Parameters:**
*   Threshold $T$: Ranges from 0 to 4095 [S2].
    *   Typical value for skin: ~400 [S2].
    *   Typical value for bones: ~1700 [S2].
    *   Typical value for teeth: 3200 [S2].
*   Light Vector $\mathbf{l}$: Moves with the view vector [S2].

**Edge Cases:**
*   Zero gradient: If $\nabla V = 0$, the normal is undefined. The method does not specify a fallback, but implementation requires a guard [G].
*   Threshold boundary: Shading is discontinuous at $T$. Voxels below $T$ receive no shading [S2].

## Subtopics

### [Diffuse Lighting for Volume Ray Casting](6760-diffuse-lighting-for-volume-ray-casting.md)
Uses gradient-based normals for Lambertian diffuse shading only. Choose when only diffuse illumination is needed.

### [Phong Shading with Boundary Enhancement and Facing Subtraction](15408-phong-shading-with-boundary-enhancement-and-facing-subtracti.md)
Modulates alpha based on gradient magnitude and view direction. Choose when silhouette enhancement and boundary effects are required.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of fragments/voxels processed. Gradient calculation requires sampling neighbors, increasing constant factor [S2].
*   **Memory Complexity:** Requires storage for scalar field and potentially precomputed gradients or on-the-fly gradient calculation [S2].
*   **Strengths:** Provides realistic 3D impression; simple to implement; interactive performance with pre-integration [S1, S2].
*   **Limitations:** Only one tissue type can be visualized at once with shading (disadvantage compared to full volume rendering) [S2]. Gradient-based normals may be noisy in low-resolution or noisy data [G].
*   **When to use:** When a realistic 3D view of a specific tissue type is needed and interactive performance is required [S2].
*   **When not to use:** When multiple tissue types with varying opacities need to be visualized simultaneously (use Volume Rendering instead) [S2].

## Background
Originated in volume visualization tools for medical imaging, specifically in interfaces allowing flexible transfer function design [S1]. The method was implemented in applications like the VOLUME RENDERING Application by Thomas Lidy, where it provided a shading option for 3D views using gradients and light vectors [S2]. It is used in medical visualization for enhancing the perception of structures like skin, bones, and teeth by applying threshold-based shading [S2].

## Sources
[S1] Volume visualization and exploration through flexible transfer function design, https://doi.org/10.1016/j.cag.2008.04.004, fetched 2026-10-27
[S2] VOLUME RENDERING Application by Thomas Lidy, https://www.cg.tuwien.ac.at/courses/Visualisierung/2001-2002/Ergebnisse/Beispiel1/LidyT/description.html, fetched 2026-10-29
