# Cook-Torrance BRDF

Also called: Microfacet BRDF, GGX Cook-Torrance, Walter et al. BRDF

## Overview

The Cook-Torrance BRDF is a physically based rendering (PBR) lighting model that uses the microfacet surface model to accurately represent real-world reflectance, distinguishing between metals and dielectrics [S1][S5, unofficial]. It solves the problem of unrealistic shading found in older models like Phong by ensuring energy conservation and modeling surface roughness via statistical microfacet alignment [S1][S2]. It sets itself apart from siblings like [Shading models](837-shading-models.md) by explicitly separating diffuse and specular lobes with energy-conserving weights and using the Fresnel effect to vary reflectivity based on viewing angle [S1][S2].

## How it works

**Inputs and Outputs:**
*   **Inputs:** Surface normal $\mathbf{n}$, view vector $\mathbf{v}$, light direction $\mathbf{l}$ (pointing from surface to light), surface albedo $c$ (color), roughness parameter $\alpha$, base reflectivity $F_0$, and light radiance $L_i$ [S1, S2].
*   **Outputs:** Outgoing radiance $L_o$ [S1, S2].

**Data Structures:**
No specific complex data structures are required beyond standard vector types for $\mathbf{n}, \mathbf{v}, \mathbf{l}, \mathbf{h}$ and scalar floats for $\alpha, F_0, c$ [G].

**Steps:**

1.  **Calculate Halfway Vector:** Compute the normalized halfway vector $\mathbf{h}$ between the light direction $\mathbf{l}$ and view direction $\mathbf{v}$ [S1, S2]:
    $$ \mathbf{h} = \frac{\mathbf{l} + \mathbf{v}}{\|\mathbf{l} + \mathbf{v}\|} $$

2.  **Compute Diffuse Weight ($k_d$) and Specular Weight ($k_s$):**
    To ensure energy conservation, the weights must sum to 1 [S1, S2]. The specular weight $k_s$ is determined by the Fresnel term at the current angle, and the diffuse weight is the remainder [S2]:
    $$ k_s = F(\mathbf{h}, \mathbf{v}, F_0) $$
    $$ k_d = 1 - k_s $$
    *Note: In many implementations, $k_s$ is treated as the specular albedo contribution, often derived from $F_0$ or the full Fresnel term [G]. The sources explicitly link $k_d$ to $(1-F)$ [S2].*

3.  **Compute Diffuse Term ($f_{Lambert}$):**
    The diffuse reflectance is the surface albedo $c$ divided by $\pi$ [S1]:
    $$ f_{Lambert} = \frac{c}{\pi} $$

4.  **Compute Normal Distribution Function ($D$):**
    Using the Trowbridge-Reitz GGX approximation, given $\mathbf{n}, \mathbf{h}$, and roughness $\alpha$ [S1, S2][S3]:
    $$ D_{GGX_{TR}}(\mathbf{n}, \mathbf{h}, \alpha) = \frac{\alpha^2}{\pi \left( (\mathbf{n} \cdot \mathbf{h})^2 (\alpha^2 - 1) + 1 \right)^2} $$

5.  **Compute Geometry Function ($G$):**
    Using Schlick-GGX with Smith's method for both view and light vectors [S1, S2][S3]:
    $$ G(\mathbf{n}, \mathbf{v}, \mathbf{l}, \alpha) = G_{Schlick}(\mathbf{n}, \mathbf{v}, k) \cdot G_{Schlick}(\mathbf{n}, \mathbf{l}, k) $$
    Where the geometry parameter $k$ is derived from roughness $\alpha$ [S1, S2][S3]:
    $$ k = \frac{(\alpha + 1)^2}{8} $$
    And the Schlick geometry term for a vector $\mathbf{i}$ (either $\mathbf{v}$ or $\mathbf{l}$) is [S1, S2][S3]:
    $$ G_{Schlick}(\mathbf{n}, \mathbf{i}, k) = \frac{\mathbf{n} \cdot \mathbf{i}}{(\mathbf{n} \cdot \mathbf{i})(1 - k) + k} $$

6.  **Compute Fresnel Function ($F$):**
    Using the Fresnel-Schlick approximation [S1, S2][S3]:
    $$ F(\mathbf{h}, \mathbf{v}, F_0) = F_0 + (1 - F_0)(1 - \mathbf{h} \cdot \mathbf{v})^5 $$
    Where $F_0$ is the base reflectivity. Default $F_0 = 0.04$ for non-metals (dielectrics) and $F_0 = c$ (albedo) for metals [S1, S2].

7.  **Compute Cook-Torrance Specular Term ($f_{CookTorrance}$):**
    $$ f_{CookTorrance} = \frac{D \cdot F \cdot G}{4 (\omega_o \cdot \mathbf{n}) (\omega_i \cdot \mathbf{n})} $$
    Where $\omega_o$ is the outgoing direction (view vector $\mathbf{v}$) and $\omega_i$ is the light direction $\mathbf{l}$ projected into the hemisphere [S2]. The dot products in the denominator are $\mathbf{n} \cdot \mathbf{v}$ and $\mathbf{n} \cdot \mathbf{l}$ [S1, S2].

8.  **Combine Terms:**
    The total BRDF $f_r$ is [S1, S2]:
    $$ f_r = k_d f_{Lambert} + k_s f_{CookTorrance} $$

9.  **Calculate Outgoing Light ($L_o$):**
    The outgoing radiance is the sum of emitted light $L_e$ and the integral of incoming light $L_i$ multiplied by the BRDF and the cosine of the incident angle [S1, S2][S3]:
    $$ L_o = L_e + \int_{\Omega} f_r L_i (\omega_i \cdot \mathbf{n}) d\omega_i $$
    For a single point light source, this simplifies to evaluating the BRDF at the light direction and multiplying by the light's radiance $L_i$ and the cosine term [S1]:
    $$ L_o = L_e + f_r L_i (\mathbf{l} \cdot \mathbf{n}) $$

10. **Light Radiance Calculation ($L_i$):**
    If using the specific polynomial attenuation model from [S1, S2]:
    $$ L_i = \left( \frac{l_{dehomogenized}}{A_{Constant} + d A_{Linear} + d^2 A_{Quadratic}} \right) \left( \frac{r - d}{r} \right) $$
    Where $l_{dehomogenized} = [l_r/l_w, l_g/l_w, l_b/l_w]$ is the light color vector, $d$ is distance, $r$ is light radius, and $A$ are attenuation coefficients [S1, S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def cook_torrance_brdf(n, v, l, albedo, roughness, F0, Li, Le):
    # 1. Normalize inputs
    n = n / math.sqrt(dot(n, n))
    v = v / math.sqrt(dot(v, v))
    l = l / math.sqrt(dot(l, l))
    
    # 2. Halfway vector
    h = (l + v) / math.sqrt(dot(l + v, l + v))
    
    # 3. Dot products (clamped to [0, 1] for shading)
    n_dot_v = max(dot(n, v), 0.0)
    n_dot_l = max(dot(n, l), 0.0)
    n_dot_h = max(dot(n, h), 0.0)
    h_dot_v = max(dot(h, v), 0.0)
    
    # 4. Geometry parameter k
    alpha = roughness # Source [S1, S2] use alpha as roughness parameter in formula
    k = ((alpha + 1.0) ** 2) / 8.0
    
    # 5. Normal Distribution Function (GGX)
    denom = (n_dot_h * n_dot_h) * (alpha * alpha - 1.0) + 1.0
    D = (alpha * alpha) / (math.pi * denom * denom)
    
    # 6. Geometry Function (Schlick-GGX)
    def schlick_ggx(n_dot_i, k):
        return n_dot_i / (n_dot_i * (1.0 - k) + k)
    
    G_v = schlick_ggx(n_dot_v, k)
    G_l = schlick_ggx(n_dot_l, k)
    G = G_v * G_l
    
    # 7. Fresnel Function (Schlick)
    F = F0 + (1.0 - F0) * ((1.0 - h_dot_v) ** 5)
    
    # 8. Specular Term
    # Denominator: 4 * (wo . n) * (wi . n)
    denom_spec = 4.0 * n_dot_v * n_dot_l
    if denom_spec < 1e-8:
        denom_spec = 1e-8 # Guard against division by zero [G]
    F_CookTorrance = (D * F * G) / denom_spec
    
    # 9. Diffuse Term
    F_Lambert = albedo / math.pi
    
    # 10. Energy Conservation Weights
    # Source [S2]: kd = 1 - F, ks = F
    ks = F
    kd = 1.0 - ks
    
    # 11. Total BRDF
    fr = kd * F_Lambert + ks * F_CookTorrance
    
    # 12. Outgoing Radiance
    # L_o = L_e + fr * Li * (l . n)
    Lo = Le + fr * Li * n_dot_l
    
    return Lo
```

**Parameters:**
*   $\alpha$ (Roughness): Controls the spread of microfacets. Typical values 0.0 (smooth) to 1.0 (rough) [S1, S2].
*   $F_0$ (Base Reflectivity): Default 0.04 for non-metals, equal to albedo for metals [S1, S2].
*   $A_{Constant}, A_{Linear}, A_{Quadratic}$: Polynomial attenuation coefficients for light distance [S1, S2].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If $\mathbf{n} \cdot \mathbf{v}$ or $\mathbf{n} \cdot \mathbf{l}$ is zero, the specular denominator becomes zero. A small epsilon guard is required [G].
*   **Energy Conservation:** If $k_d$ and $k_s$ are not properly linked (e.g., $k_d = 1 - k_s$), energy may not be conserved, leading to overly bright surfaces [S1, S2].
*   **Roughness Remapping:** Source [S1] defines $\alpha = \text{roughness}^2$, while [S2] uses $\alpha$ directly as the roughness parameter in the formula structure. Implementers must check which convention the rest of the pipeline uses [S1][S3].

## Subtopics

### Cook-Torrance GGX BRDF
A variant that combines Cook-Torrance specular with Lambertian diffuse, featuring an anisotropic GGX normal distribution function, Smith height-correlated geometry term, and Schlick Fresnel approximation [S7, unofficial]. It applies Fresnel-weighted energy conservation between diffuse and specular lobes and uses roughness-aware Fresnel for environment map reflections [S7, unofficial]. It is chosen when anisotropic reflections are required, distinguishing it from the isotropic standard Cook-Torrance model [S7, unofficial].

## Operating characteristics

*   **Time Complexity:** $O(1)$ per light source per pixel/voxel, as it involves a fixed number of trigonometric and arithmetic operations [G].
*   **Memory Complexity:** $O(1)$, requiring only a few temporary vectors and scalars [G].
*   **Measured Performance:** Not given in the sources [G].
*   **Strengths:** Physically accurate, energy-conserving, handles a wide range of materials (metals/dielectrics) and roughness levels [S1][S3].
*   **Limitations:** More computationally expensive than Phong or Lambertian models [S1]. Requires careful handling of energy conservation [S1, S2].
*   **When to Use:** When realistic material appearance is required, especially for metals and rough surfaces [S1][S3].

## Background

The Cook-Torrance BRDF was introduced by Richard L. Cook and Kenneth E. Torrance in 1982 [S1][S3]. It is based on the microfacet surface model, which approximates surface roughness statistically [S1][S3]. It is widely used in Physically Based Rendering (PBR) pipelines to replace older models like Phong, which fail to conserve energy [S1][S3].

## Sources

[S1] Deep Reflectance Volumes: Relightable Reconstructions from Multi-view Photometric Images, https://doi.org/10.1007/978-3-030-58580-8_18, fetched 2026-10-26
[S2] Raymarching Distance Fields with CUDA, https://doi.org/10.3390/electronics10222730, fetched 2026-10-26
[S3] Jrender: An efficient differentiable rendering library based on Jittor, https://www.sciencedirect.com/science/article/pii/S1524070323000322, fetched 2026-11-01
[S4] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S5] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S6] (PDF) Raymarching Distance Fields with CUDA - ResearchGate, https://www.researchgate.net/publication/356081826_Raymarching_Distance_Fields_with_CUDA, fetched 2026-10-27 (unofficial)
[S7] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
