# Shading models

Also called: Diffuse and Specular Shading for Volume Rendering

## Overview
Shading models for volume rendering apply local illumination calculations to scalar field data to enhance the perception of 3D shape, surface detail, and orientation [S1, unofficial]. Unlike methods that treat volumes purely as participating media for light transport (e.g., [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md)) or frequency-domain approximations (e.g., [Frequency Domain Directional Shading](852-frequency-domain-directional-shading.md)), this method treats the volume as a collection of pseudo-surfaces, computing diffuse and specular contributions based on data values and their gradients within a ray-casting framework [S1, unofficial]. It distinguishes itself from [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md) by explicitly managing the interplay between material opacity and light transport modes (indirect vs. direct) within a single shading pass, and from [Lambert Shading with Shadow Rays](1133-lambert-shading-with-shadow-rays.md) by including specular highlights [S1, S11, unofficial].

## How it works
**Inputs:**
*   Scalar volume data $V$ (e.g., trilinearly interpolated from a grid).
*   Gradient data $\nabla V$ (derived from the derivative of the interpolation function or finite differences) [S1, unofficial].
*   Transfer function $TF(v)$ mapping data values to colors [S1, unofficial].
*   Shading parameters:
    *   $k_d$: Amount of diffuse shading [S1, unofficial].
    *   $k_s$: Amount of specular shading [S1, unofficial].
    *   $\alpha$: Specular shininess exponent [S1, unofficial].
    *   $\tau_{grad}$: Minimum gradient magnitude threshold [S1, unofficial].
    *   $[v_{min}, v_{max}]$: Shading range for data values [S1, unofficial].
    *   $\alpha_{shade}$: Opacity of the shading contribution [S1, unofficial].
*   Light direction $\mathbf{l}$ (assumed directional for standard implementation) [G].
*   View direction $\mathbf{v}$ [G].

**Outputs:**
*   Shaded color contribution $C_{shade}$ for the current ray segment or intersection point.

**Data Structures:**
*   **Ray-Region Intersection Data:** For each ray traversing a region of interest, store:
    *   $v_{entry}, v_{exit}$: Data values at entry and exit points of the region.
    *   $\mathbf{g}_{entry}, \mathbf{g}_{exit}$: Gradient vectors at entry and exit points.
    *   $\|\mathbf{g}_{entry}\|, \|\mathbf{g}_{exit}\|$: Magnitudes of the gradients.

**Steps:**

1.  **Evaluate Intersection Properties:**
    For a ray passing through a region, determine the data values $v_{entry}, v_{exit}$ and gradients $\mathbf{g}_{entry}, \mathbf{g}_{exit}$ at the entry and exit boundaries of the region. The gradient is typically computed as the derivative of the trilinear interpolation function used for volume rendering [S1, unofficial].

2.  **Check Shading Eligibility:**
    Shading is applied to the region if:
    *   Either $v_{entry}$ or $v_{exit}$ lies within the shading range $[v_{min}, v_{max}]$.
    *   AND either $\|\mathbf{g}_{entry}\|$ or $\|\mathbf{g}_{exit}\|$ is greater than or equal to $\tau_{grad}$ [S1, unofficial].
    If neither condition is met, the shading contribution for this region is zero [S1, unofficial].

3.  **Compute Average Data Value for Diffuse Reflectivity:**
    If eligible, calculate the effective data value $v_{shade}$ for diffuse reflection:
    $$ v_{shade} = \frac{v_{entry} + v_{exit}}{2} $$
    This value determines the diffuse color via the transfer function: $C_{diffuse\_base} = TF(v_{shade})$ [S1, unofficial].

4.  **Compute Average Gradient for Normal:**
    Calculate the effective normal vector $\mathbf{n}$:
    *   If both $\|\mathbf{g}_{entry}\|$ and $\|\mathbf{g}_{exit}\|$ are within valid ranges (i.e., $\ge \tau_{grad}$):
        $$ \mathbf{g}_{avg} = \frac{\mathbf{g}_{entry} + \mathbf{g}_{exit}}{2} $$
    *   If only one gradient magnitude is within range (e.g., $\|\mathbf{g}_{entry}\| \ge \tau_{grad}$ but $\|\mathbf{g}_{exit}\| < \tau_{grad}$):
        $$ \mathbf{g}_{avg} = \mathbf{g}_{entry} $$
    *   If neither is within range, shading is not applied (handled in Step 2).
    
    Normalize the gradient to obtain the surface normal:
    $$ \mathbf{n} = \frac{\mathbf{g}_{avg}}{\|\mathbf{g}_{avg}\|} $$
    [S1, unofficial].

5.  **Calculate Diffuse Contribution:**
    Compute the Lambertian diffuse term:
    $$ C_{diff} = k_d \cdot \max(0.0, \mathbf{n} \cdot \mathbf{l}) \cdot C_{diffuse\_base} $$
    [S1, unofficial]. The diffuse reflection color is assumed to be the color from the transfer function at $v_{shade}$ [S1, unofficial].

6.  **Calculate Specular Contribution:**
    Compute the specular term using the Phong model. The specular reflection is assumed to be white ($C_{specular\_base} = \mathbf{1}$) [S1, unofficial].
    $$ \mathbf{h} = \frac{\mathbf{l} + \mathbf{v}}{\|\mathbf{l} + \mathbf{v}\|} $$
    $$ C_{spec} = k_s \cdot (\max(0.0, \mathbf{n} \cdot \mathbf{h}))^\alpha $$
    [S1, unofficial].

7.  **Combine Shading Contributions:**
    The total shading color $C_{shade}$ is the sum of diffuse and specular components, modulated by the shading opacity:
    $$ C_{shade} = \alpha_{shade} \cdot (C_{diff} + C_{spec}) $$
    [S1, unofficial].

8.  **Integrate into Volume Rendering:**
    The shading contribution $C_{shade}$ is typically accumulated along the ray, often blended with the volume's inherent color and opacity based on the specific ray-casting implementation (e.g., additive or multiplicative blending with the transfer function color) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Gradient Discontinuity:** Using the derivative of the trilinear function for gradients can cause discontinuities at shared cell faces, potentially introducing shading artifacts [S1, unofficial]. Finite differences may be used as an alternative but are more computationally expensive [S1, unofficial].
*   **Zero Gradient:** If $\|\mathbf{g}_{avg}\|$ is zero, the normal is undefined. This is handled by the threshold check in Step 2; if the gradient magnitude is below $\tau_{grad}$, shading is skipped.
*   **Out-of-Range Gradients:** The logic for selecting the gradient when one is out of range must be carefully implemented to avoid using invalid normals [S1, unofficial].

## Subtopics

### [Frequency Domain Directional Shading](852-frequency-domain-directional-shading.md)
Frequency Domain Directional Shading is a local illumination technique for volume rendering that computes diffuse shading entirely in the frequency domain, avoiding spatial domain differentiation. It solves the problem of applying directional lighting cues to volume data without the computational cost or nonlinearity issues of spatial domain methods. Unlike [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md) and [Lambert Shading with Shadow Rays](1133-lambert-shading-with-shadow-rays.md), which operate in the spatial domain or require shadow rays, this method leverages the property that spatial differentiation corresponds to multiplication by a linear ramp in the frequency domain. It differs from [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md) by using a hemispherical light source model to maintain linearity, enabling efficient frequency domain computation.

### [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md)
The Simplified Phong Lighting Model is a local illumination technique used in volume rendering to enhance the perception of 3D shape and surface detail in scalar fields. It solves the problem of flat, ambiguous volumetric data by computing diffuse and specular lighting contributions based on estimated surface normals derived from volume gradients. Unlike [Frequency Domain Directional Shading](852-frequency-domain-directional-shading.md), which operates in the Fourier domain, or [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md), which models light scattering within the medium, this method treats the volume as a collection of pseudo-surfaces for local shading purposes. It differs from [Lambert Shading with Shadow Rays](1133-lambert-shading-with-shadow-rays.md) by typically omitting shadow calculations to maintain performance, and from the [Volume Shading Model](2991-volume-shading-model.md) by focusing on surface-like reflection properties rather than participating media absorption and emission.

### [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md)
Gradient-based shading with phase functions adds an illumination term to the emissive term of the volume rendering integral to simulate single-scattering effects, or computes shading for implicit iso-surfaces. It distinguishes itself from sibling methods by explicitly deriving surface normals from the scalar field gradient and applying phase functions (such as Lambert's law or Henyey-Greenstein) to model light interaction, rather than relying solely on frequency domain approximations or simplified Phong models.

### [Lambert Shading with Shadow Rays](1133-lambert-shading-with-shadow-rays.md)
Lambert Shading with Shadow Rays is a local illumination technique for isosurface rendering that computes diffuse lighting using the Lambertian reflectance model and enhances depth perception by casting shadow rays to detect occlusion. It distinguishes itself from sibling models like [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md) by focusing exclusively on diffuse reflection without specular highlights, and from [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md) by treating the volume as an opaque isosurface rather than a participating medium for light transport.

### [Volume Shading Model](2991-volume-shading-model.md)
The Volume Shading Model is a local illumination technique implemented within ray casting volume renderers that calculates diffuse and specular contributions based on scalar data values and their gradients. It computes the final fragment color by combining indirect diffuse, indirect specular, emissivity, transparency/refraction, and direct light contributions, applying shadow blending and gamma correction. It differs from its siblings by explicitly managing the interplay between material opacity and light transport modes (indirect vs. direct) within a single shading pass, whereas [Simplified Phong Lighting Model](885-simplified-phong-lighting-model.md) focuses on surface-like reflection approximations and [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md) emphasizes angular scattering properties.

### Fire Shading Model
A shading model that combines Lambertian diffuse lighting, ambient light, and rim lighting, modulated by a fire-colored palette driven by animated fBM noise [S2, unofficial]. It computes the surface normal `N` using `distance_field_normal(hit)`, the light direction `L` normalized from the hit point to the light position, and the view direction `V` normalized from the hit point to the camera position [S2, unofficial]. The Lambertian diffuse factor is `max(0.0, dot(N, L))` and the rim light factor is `pow(max(0.0, 1.0 - max(0.0, dot(N, V))), 2.0)` [S2, unofficial]. The total light intensity is `ambient + 0.9 * lambert + 0.4 * rim` [S2, unofficial]. The fire color value `fval` is computed by evaluating fBM at the hit point with a time offset, mapped to a fire color using `palette_fire(fval)` [S2, unofficial]. The final color is `fire * light_intensity` [S2, unofficial]. The parameter `ambient` is a constant ambient light factor [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples along the ray, as shading is computed per sample or per region intersection [S1, unofficial]. The gradient computation adds a constant overhead per sample if using analytical derivatives of trilinear interpolation [S1, unofficial].
*   **Memory Complexity:** Low, as it only requires storing the current ray's intersection data and shading parameters [S1, unofficial].
*   **Performance:** Shading helps considerably in bringing out features in the data and establishing orientations, as observed in comparisons with and without shading [S1, unofficial]. However, it can be computationally more expensive than unshaded volume rendering due to gradient calculations and additional lighting terms [S1, unofficial].
*   **Strengths:** Enhances visual perception of 3D structure and surface details; simple to implement within existing ray-casting frameworks; provides intuitive visual cues for orientation [S1, unofficial].
*   **Limitations:** Does not account for complex light transport effects like subsurface scattering or multiple scattering; gradient discontinuities can cause artifacts; performance may be impacted by frequent gradient calculations [S1, unofficial].
*   **When to Use:** When the goal is to enhance the visual clarity of scalar field features and surface-like structures in volume data, and when computational resources allow for per-sample shading calculations [S1, unofficial].
*   **When Not to Use:** When realistic light transport in participating media is required (use [Gradient-based Shading with Phase Functions](1020-gradient-based-shading-with-phase-functions.md) or similar), or when performance is critical and shading artifacts are unacceptable [S1, unofficial].

## Background
The method was implemented within a ray-casting volume renderer, as described in early work on hierarchically accelerated ray casting for volume rendering [S1, unofficial]. The implementation adopted multiprocessing for parallelization on SGI workstations [S1, unofficial]. The approach was compared with other rendering methods such as 3D texture mapping and coherent projection, highlighting the advantages of ray casting in accommodating shading and producing crisp edges around features [S1, unofficial]. The technique has been referenced in various contexts, including educational materials and informal discussions on volume rendering techniques [S1, unofficial][S2, unofficial].

## Sources
[S1] [PDF] Hierarchically Accelerated Ray Casting for Volume Rendering with ..., https://tr.soe.ucsc.edu/sites/default/files/technical-reports/UCSC-CRL-95-31.pdf, fetched 2026-10-27 (unofficial)
[S2] OmerNYU/Kaboom_Diffusion, https://github.com/OmerNYU/Kaboom_Diffusion, fetched 2026-10-26 (unofficial)
