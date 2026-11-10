# Relightable 3DGS

Also called: Handling Challenging Objects (Reflections/Transparency), Relightable 3D GS, Specular 3D GS

## Overview
Relightable 3DGS extends standard 3D Gaussian Splatting by embedding physically-based material properties into each Gaussian, enabling deferred shading and realistic rendering of complex appearances such as reflections, transparency, and glossy surfaces [S1, unofficial][S2]. Unlike standard rasterization which approximates view-dependent colors via spherical harmonics and fails to represent specular reflections or global illumination, this method decomposes outgoing radiance into a BRDF integration that respects material parameters [S1, unofficial]. It sets itself apart from its siblings by treating alpha-blending as a smoothing filter to stabilize optimization while allowing for physically accurate lighting interactions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\{G_i\}$, each defined by position, covariance (or scale/orientation), opacity, and now including physically-based material properties (albedo, metallic, roughness, normal). Incident lighting environment $L_i(\omega_i)$ and view direction $\omega_0$.
*   **Outputs:** The final pixel color $c'$, computed via deferred shading.

**Data Structures:**
Each Gaussian $i$ stores:
*   $\mathbf{p}_i$: 3D position.
*   $\mathbf{\Sigma}_i$: Covariance matrix (or scale $s$ and rotation $q$).
*   $\alpha_i$: Opacity.
*   $\mathbf{n}_i$: Surface normal.
*   Material parameters: Albedo $\rho_d$, Metallic $m$, Roughness $r$.
*   (Optional) View-dependent features for anisotropic components [S2].

**Steps:**
1.  **Rasterization:** Project each Gaussian to screen space. Compute the 2D covariance matrix $\mathbf{\Sigma}'$ and screen-space center $\mathbf{c}$.
2.  **Alpha Blending of Features:** Aggregate per-Gaussian material properties (albedo, metallic, roughness, normal) into pixel-level feature maps via alpha-blending. This step treats alpha-blending as a smoothing filter to stabilize optimization and produce cohesive results [S1, unofficial].
    *   For a pixel $p$, the accumulated feature $F_p$ is computed by sorting Gaussians by depth and blending:
        $$ F_p = \sum_{i \in \text{sorted}} \alpha_i \prod_{j < i} (1 - \alpha_j) \cdot \text{Material}_i $$
3.  **Deferred Shading (BRDF Integration):** For each pixel, evaluate the rendering equation using the accumulated material properties and the lighting environment.
    *   Compute the specular BRDF using the microfacet Cook-Torrance model [S1, unofficial]:
        $$ f_s(\omega_0, \omega_i) = \frac{D(h; r) \cdot F(\omega_0, h) \cdot G(\omega_i, \omega_0, h; r)}{(n \cdot \omega_i) \cdot (n \cdot \omega_0)} $$
        where:
        *   $\omega_0$: View direction.
        *   $\omega_i$: Incident lighting direction.
        *   $h = (\omega_i + \omega_0)/2$: Half vector.
        *   $D$: Normal distribution function (NDF).
        *   $F$: Fresnel term.
        *   $G$: Geometry term (shadowing/masking).
        *   $n$: Normal vector.
        *   $r$: Roughness.
    *   Compute the final PBR color $c'(\omega_0)$ by integrating over the incident lighting directions $\omega_i$ [S1, unofficial]:
        $$ c'(\omega_0) = \sum_{i=0}^{N_s} (f_d + f_s(\omega_0, \omega_i)) L_i(\omega_i) (\omega_i \cdot n) \Delta \omega_i $$
        where:
        *   $f_d$: Diffuse component (typically $\rho_d / \pi$).
        *   $L_i(\omega_i)$: Incident radiance from direction $\omega_i$.
        *   $\Delta \omega_i$: Solid angle associated with the sampled incident direction.
4.  **Shadow Casting (for Point Sources):** To compute direct illumination from a point source, trace secondary rays from each Gaussian to the light source and evaluate a shadow map [S1, unofficial].

**Parameters:**
*   $N_s$: Number of light samples. Not given in the sources.
*   Roughness $r$: Typically optimized or provided as input. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** Alpha-blending of material properties requires careful handling of opacity accumulation to avoid artifacts. [G] Ensure that the product of $(1-\alpha_j)$ is computed in a numerically stable way (e.g., using log-space or iterative multiplication) to prevent underflow for large numbers of Gaussians.
*   **Boundaries:** The method struggles with robust reconstruction of reflective and transparent objects in general scenes [S2].
*   **Degenerate Input:** If a Gaussian has no defined normal or material properties, the BRDF evaluation may fail. [G] Default to a diffuse Lambertian model or skip shading for such Gaussians.

## Subtopics

### [Deferred Shading with Alpha-Blended Feature Maps](12306-deferred-shading-with-alpha-blended-feature-maps.md)
This method renders relightable 3D Gaussians by aggregating per-Gaussian material properties (albedo, metallic, roughness, normal) into pixel-level feature maps via alpha-blending, then applying the rendering equation in a deferred shading pass. Unlike shading directly on Gaussians before blending, this approach treats alpha-blending as a smoothing filter to stabilize optimization and produce cohesive results.

## Operating characteristics
*   **Time Complexity:** Real-time performance during view synthesis is preserved [S1, unofficial]. The deferred shading pass adds computational cost proportional to the number of light samples $N_s$ and the complexity of the BRDF evaluation.
*   **Memory Complexity:** Increased memory usage due to storing material properties (albedo, metallic, roughness, normal) for each Gaussian, in addition to position, covariance, and opacity.
*   **Strengths:** Enables relighting the entire scene under novel illumination conditions; handles complex appearances like reflective, transparent, highly glossy, or anisotropic surfaces better than standard 3D GS [S1, unofficial][S2].
*   **Limitations:** Robust reconstruction of reflective and transparent objects in general scenes remains challenging [S2]. Inherits difficulties from 3D GS regarding approximations that limit illumination estimation accuracy [S2].
*   **When to use:** When physically-based relighting and accurate specular reflections are required.
*   **When not to use:** When real-time performance is critical and lighting changes are not needed, or when dealing with highly complex transparent scenes where robust reconstruction is difficult.

## Background
Standard 3D Gaussian Splatting uses a view-dependent spherical harmonics color model, which fails to represent specular reflections, glossy materials, and global illumination [S1, unofficial]. Relightable 3DGS represents a major paradigm shift by embedding physically-based material properties into each Gaussian for deferred shading [S1, unofficial]. It decomposes the outgoing radiance into a BRDF integration that respects material parameters [S1, unofficial]. Related approaches include GaussianShader, which simplifies the BRDF to a neural shader, and GS-IR, which performs inverse rendering by jointly optimizing Gaussian geometry and an environment light probe [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
