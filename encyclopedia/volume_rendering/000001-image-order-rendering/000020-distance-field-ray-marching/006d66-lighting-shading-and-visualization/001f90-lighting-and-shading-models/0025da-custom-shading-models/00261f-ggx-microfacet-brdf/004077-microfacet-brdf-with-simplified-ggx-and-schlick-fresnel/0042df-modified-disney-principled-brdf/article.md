# Modified Disney Principled BRDF

Also called: ABSDF Base Material, Base Material Model

## Overview
The Modified Disney Principled BRDF is a simplified surface reflectance model derived from the Disney Principled BRDF, tailored for use in the Absorption-Based Scene Diffusion Factorization (ABSDF) framework [S1, unofficial]. It solves the problem of efficiently aggregating complex microfacet reflections by reducing the original model's complexity while retaining its core appearance capabilities for metals and plastics [S1, unofficial]. It sets itself apart from its parent, the [Microfacet BRDF with Simplified GGX and Schlick Fresnel](10259-microfacet-brdf-with-simplified-ggx-and-schlick-fresnel.md), by explicitly adopting a Lambertian diffuse term, omitting sheen and clearcoat lobes, and assuming double-sided surfaces to facilitate analytical factorization [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Incoming light direction $\omega_i$, outgoing view direction $\omega_o$, surface normal $n$, half vector $\omega_h = \frac{\omega_i + \omega_o}{\|\omega_i + \omega_o\|}$, and material parameters $\beta_B = (\alpha, \beta_c, \beta_m, \beta_s)$ [S1, unofficial].
*   **Outputs:** Bidirectional Reflectance Distribution Function (BRDF) value $f_{disney}(\omega_i, \omega_o)$ [S1, unofficial].

**Data Structures:**
The material parameters $\beta_B$ are spatially varying scalars [S1, unofficial]:
*   $\alpha \in [0, 1]$: Roughness.
*   $\beta_c \in [0, 1]$: Base color (diffuse albedo).
*   $\beta_m \in [0, 1]$: Metallic factor.
*   $\beta_s \in [0, 1]$: Specular intensity (dielectric reflectance at normal incidence) [S1, unofficial].

**Formulas:**
The model is defined as the sum of a diffuse component $f_d$ and a specular component $f_s$ [S1, unofficial]:

$$
f_{disney}(\omega_i, \omega_o) = f_d(\omega_i, \omega_o) + f_s(\omega_i, \omega_o)
$$

1.  **Diffuse Component:** Uses the Lambertian model instead of the original empirical retro-reflection term [S1, unofficial]:
    $$
    f_d(\omega_i, \omega_o) = \frac{(1 - \beta_m)\beta_c}{\pi}
    $$
    Here, $\beta_m$ reduces the diffuse contribution for metallic surfaces, and $\beta_c$ is the base color albedo [S1, unofficial].

2.  **Specular Component:** Combines metallic and dielectric Fresnel reflections using the GGX distribution $D$, the shadowing-masking function $G$, and the Schlick Fresnel function $F$ [S1, unofficial]:
    $$
    f_s(\omega_i, \omega_o) = \frac{\beta_m D(\omega_h; \alpha) G(\omega_i, \omega_o; \alpha) F(\omega_h, \omega_o; \beta_c) + (1 - \beta_m) F(\omega_h, \omega_o; \beta_s)}{4 |n \cdot \omega_o| |n \cdot \omega_i|}
    $$
    *   $D(\omega_h; \alpha)$: Trowbridge-Reitz (GGX) microfacet distribution.
    *   $G(\omega_i, \omega_o; \alpha)$: Shadowing-masking function.
    *   $F(\omega_h, \omega_o; r_0)$: Schlick Fresnel reflectance.

3.  **Schlick Fresnel:**
    $$
    F(\omega_h, \omega_o; r_0) = r_0 (1 - F_c) + F_c
    $$
    $$
    F_c = (1 - |\omega_h \cdot \omega_o|)^5
    $$
    Where $r_0$ is the normal incidence reflectance [S1, unofficial]. For the metallic lobe, $r_0 = \beta_c$. For the dielectric lobe, $r_0 = \beta_s$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def modified_disney_brdf(omega_i, omega_o, n, alpha, beta_c, beta_m, beta_s):
    PI = 3.141592653589793
    omega_i = omega_i / norm(omega_i)
    omega_o = omega_o / norm(omega_o)
    n = n / norm(n)
    
    omega_h = (omega_i + omega_o) / norm(omega_i + omega_o)
    
    cos_theta_i = dot(n, omega_i)
    cos_theta_o = dot(n, omega_o)
    cos_theta_h = dot(n, omega_h)
    
    if cos_theta_i <= 0.0 or cos_theta_o <= 0.0:
        return 0.0
        
    denom = 4.0 * cos_theta_i * cos_theta_o
    
    # 1. Diffuse Component (Lambertian)
    f_diffuse = ((1.0 - beta_m) * beta_c) / PI
    
    # 2. Specular Component
    # D: Trowbridge-Reitz (GGX) distribution [G]
    alpha_sq = alpha * alpha
    denom_D = (cos_theta_h * cos_theta_h) * (alpha_sq - 1.0) + 1.0
    D = alpha_sq / (PI * denom_D * denom_D)
    
    # G: Schlick-GGX shadowing-masking function [G]
    k = (alpha + 1.0) * (alpha + 1.0) / 8.0
    G_i = cos_theta_i / (cos_theta_i * (1.0 - k) + k)
    G_o = cos_theta_o / (cos_theta_o * (1.0 - k) + k)
    G = G_i * G_o
    
    # Schlick Fresnel
    cos_h_o = abs(dot(omega_h, omega_o))
    F_c = (1.0 - cos_h_o) ** 5.0
    
    # Metallic lobe Fresnel (r0 = beta_c)
    F_metal = beta_c * (1.0 - F_c) + F_c
    
    # Dielectric lobe Fresnel (r0 = beta_s)
    F_dielectric = beta_s * (1.0 - F_c) + F_c
    
    # Specular term
    f_specular = (beta_m * D * G * F_metal + (1.0 - beta_m) * F_dielectric) / denom
    
    return f_diffuse + f_specular
```

**Edge Cases and Pitfalls:**
*   **Double-Sided Assumption:** The model assumes double-sided surfaces, meaning the BRDF is evaluated regardless of the sign of $n \cdot \omega_i$ or $n \cdot \omega_o$ in the context of the factorization integral, though the geometric terms $|n \cdot \omega|$ handle the dot product magnitude [S1, unofficial].
*   **Retro-reflection Omission:** The original Disney model includes a retro-reflection term for diffuse light; this is omitted here, which may cause slight appearance differences for grazing angles on rough surfaces [S1, unofficial].

## Subtopics

### [Diffuse ABSDF Factorization](10870-diffuse-absdf-factorization.md)
A method for approximating the diffuse component of light transport in participating media by separating integration over material parameters from integration over geometry and orientation. It sets itself apart from sibling methods by focusing specifically on the diffuse lobe, allowing the use of Spherical Gaussian approximations for the cosine-weighted normal distribution. Choose it when efficiently evaluating the diffuse scattering term in path tracing is required.

### [Specular ABSDF Factorization](10872-specular-absdf-factorization.md)
A method that computes the aggregated specular response of a surface with spatially varying material parameters by factorizing the integral of the microfacet BRDF over a joint probability distribution of those parameters. It sets itself apart by explicitly separating Fresnel mix terms from the microfacet distribution convolution and applying the shadowing-masking term post-convolution. Choose it when evaluating the expectation of a complex microfacet distribution under orientation-varying parameters is required.

### [Phys3DGS BRDF Function](11071-phys3dgs-brdf-function.md)
A physically-based BRDF function used in 3D Gaussian Splatting for inverse rendering, consisting of a diffuse and a specular component. It sets itself apart from its parent by employing a specific roughness remapping ($\alpha = r^2$) and a correlated Smith masking-shadowing function rather than a separable approximation. Choose it when higher physical accuracy is needed for Gaussian splatting optimization.

### [Isotropic Disney BRDF](14196-isotropic-disney-brdf.md)
A simplified material model based on the Cook–Torrance microfacet framework, designed to model reflectance for both metals and plastics in a unified framework. It sets itself apart by restricting optimization to opaque surfaces with spatially varying diffuse albedo and global constants for roughness, specular albedo, and metalness. Choose it for differentiable rendering and optimization tasks where ambiguity from high-dimensional parameter spaces must be reduced.

## Operating characteristics
*   **Complexity:** The base evaluation is $O(1)$ per sample. The ABSDF factorization allows for pre-computation of moments, reducing runtime aggregation costs for level-of-detail rendering [S1, unofficial].
*   **Strengths:** Simplicity compared to the full Disney BRDF; compatibility with analytical factorization techniques; wide applicability to standard PBR assets [S1, unofficial].
*   **Limitations:** Lack of sheen and clearcoat lobes limits the ability to model certain materials (e.g., hair, varnished wood) accurately [S1, unofficial]. The Lambertian diffuse approximation may not capture subsurface scattering effects or retro-reflection present in the original model [S1, unofficial].
*   **When to use:** When using the ABSDF framework for efficient scene aggregation or when a simplified, robust PBR model is needed for differentiable rendering [S1, unofficial].

## Background
The Modified Disney Principled BRDF was introduced in the context of the Efficient Scene Appearance Aggregation for Level-of-Detail Rendering paper [S1, unofficial]. It is based on the Disney Principled BRDF [Burley 2012], which is one of the most commonly used models in production for its versatility [S1, unofficial]. The modifications were made to facilitate the mathematical factorization required for the ABSDF method, specifically by simplifying the diffuse term and removing complex lobes [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
