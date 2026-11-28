# Per-Point Normal Lighting

Also called: Gradient Normal Lighting, Normal-vector-based lighting model

## Overview
Per-Point Normal Lighting is a volume rendering technique that calculates the surface normal vector at each individual resampling point along the ray and substitutes this normal directly into the lighting model [S1]. This approach solves the problem of visual flatness and loss of microstructural detail inherent in methods that interpolate lighting results without computing per-sample normals [S1]. By computing normals at the sampling points, the method accurately represents diffuse and specular reflections, preserving the three-dimensional sense and concavity/convexity of the data field [S1].

## How it works
The method operates on a scalar field $V$ defined over a 3D grid. The core distinction from its parent, [Diffuse Lighting for Volume Ray Casting](6760-diffuse-lighting-for-volume-ray-casting.md), is the explicit calculation of the gradient at every resampling point rather than relying on interpolated or pre-computed normals.

**Inputs and Outputs:**
*   **Input:** Scalar field $V(x,y,z)$, viewing direction $\mathbf{v}$, light direction $\mathbf{l}$, camera position, and resampling points along the ray.
*   **Output:** Final color $C$ for each pixel, incorporating diffuse and specular components calculated using per-point normals.

**Data Structures:**
*   **Scalar Field:** A 3D array of floating-point values representing the volume density or intensity.
*   **Resampling Point:** A 3D coordinate $(x,y,z)$ along the ray, with an associated scalar value $V$ and computed normal $\mathbf{n}$.

**Steps:**
1.  **Ray Marching:** Cast rays through the volume. For each ray, determine a sequence of resampling points $P_i$ at positions $\mathbf{p}_i$.
2.  **Normal Calculation:** For each resampling point $\mathbf{p}_i$, calculate the normal vector $\mathbf{n}_i$ using finite differences of the scalar field $V$. The normal is the gradient of the scalar field:
    $$ \mathbf{n}_i = \nabla V(\mathbf{p}_i) = \left[ \frac{\partial V}{\partial x}, \frac{\partial V}{\partial y}, \frac{\partial V}{\partial z} \right]_{\mathbf{p}_i} $$
    [G] In practice, central differences are used for accuracy: $\frac{\partial V}{\partial x} \approx \frac{V(x+h, y, z) - V(x-h, y, z)}{2h}$.
3.  **Lighting Model Application:** Substitute $\mathbf{n}_i$ into the lighting model. The diffuse component $L_d$ and specular component $L_s$ are calculated using the standard Phong or Blinn-Phong equations, where $\mathbf{n}_i$ is normalized:
    $$ L_d = k_d (\mathbf{n}_i \cdot \mathbf{l}) $$
    $$ L_s = k_s (\max(0, \mathbf{r} \cdot \mathbf{v}))^{\alpha} $$
    where $\mathbf{r}$ is the reflection vector derived from $\mathbf{n}_i$, and $k_d, k_s, \alpha$ are material constants.
4.  **Compositing:** Accumulate the color contributions from all resampling points along the ray, typically using front-to-back alpha compositing.
5.  **Contour Smoothing:** Render with a slightly fuzzy contour line to avoid jaggedness and distortion [S1]. This is often achieved by blending the lighting result with a threshold-based edge detection or by using a soft threshold in the transfer function.

**Parameters:**
*   **Step Size:** Distance between resampling points. Smaller steps increase accuracy but cost more.
*   **Threshold:** Value used to define the surface boundary for normal calculation and compositing.
*   **Fuzziness:** Degree of blurring applied to the contour to mitigate aliasing [S1].

**Edge Cases and Pitfalls:**
*   **Jaggedness:** Without the fuzzy contour step, discrete sampling can cause jagged edges [S1].
*   **Noise:** Finite difference normal calculation is sensitive to noise in the scalar field, which can lead to erratic lighting. Smoothing the field before gradient calculation is often necessary.
*   **Flat Surfaces:** If normals are not calculated per point but interpolated, surfaces appear flat and lose concavity/convexity [S1].

## Subtopics

### Mesh Lighting
A shading technique for mesh displays that varies brightness based on the angle between the surface normal and the line of sight, making the inside of the volume dimmer than the outside [S2]. Brightness is maximal when the outward-facing normal is parallel to the line of sight and pointing at the user [S2]. The definition of "outward" depends on the `flipNormals` setting and the threshold sign: if `flipNormals` is true, the side toward larger values is outside for negative thresholds, and the side toward smaller values is outside for positive thresholds [S2].

## Operating characteristics
*   **Accuracy:** Maintains high rendering accuracy for microstructures and preserves 3D sense better than methods that interpolate lighting results [S1].
*   **Performance:** Requires calculating gradients at every resampling point, which is more computationally expensive than pre-computed or interpolated normals.
*   **Visual Quality:** Avoids the loss of concavity/convexity and the "flat surface" appearance associated with current interpolation methods [S1].
*   **Limitations:** Sensitive to noise in the scalar field; requires careful handling of contour fuzziness to avoid distortion [S1].

## Background
The method was introduced to address the limitations of existing volume rendering techniques that interpolate lighting model results, which fail to accurately represent diffuse and specular reflections at sampling points [S1]. It was evaluated on explosion field features, including projectile penetration into reinforced concrete and aerial explosions, demonstrating superior preservation of microstructural details [S1]. The approach is distinct from mesh-based lighting techniques like Mesh Lighting, which apply shading to surface meshes rather than volumetric resampling points [S2].

## Sources
[S1] A volume rendering visualization method oriented to explosion field features, https://doi.org/10.1063/5.0226283, fetched 2026-10-26
[S2] Command: volume, https://www.rbvi.ucsf.edu/chimerax/docs/user/commands/volume.html, fetched 2026-10-27
