# Modified Disney Principled BRDF

Also called: ABSDF Base Material, Base Material Model

## Overview
A simplified material model based on the Disney Principled BRDF [Burley 2012], modified specifically to enable efficient Aggregated Bidirectional Scattering Distribution Function (ABSDF) factorization [S1, unofficial]. It solves the problem of rendering heterogeneous materials with varying roughness and orientation by stripping the original model down to its core diffuse and specular components, allowing for separable integral approximations. It is distinguished from its parent, [Microfacet BRDF with Simplified GGX and Schlick Fresnel](10259-microfacet-brdf-with-simplified-ggx-and-schlick-fresnel.md), by explicitly adopting a Lambertian diffuse term and omitting sheen/clearcoat lobes to facilitate the statistical aggregation required for ABSDF [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Incident direction $\omega_i$, outgoing direction $\omega_o$, surface normal $n$, half vector $\omega_h = \frac{\omega_i + \omega_o}{\|\omega_i + \omega_o\|}$, and spatially-varying material parameters $\beta_B = (\alpha, \beta_c, \beta_m, \beta_s)$.
    *   $\alpha$: Roughness.
    *   $\beta_c$: Base color (diffuse albedo).
    *   $\beta_m$: Metallic factor.
    *   $\beta_s$: Specular intensity (dielectric reflectance).
*   **Output:** Bidirectional Reflectance Distribution Function (BRDF) value $f_{disney}(\omega_i, \omega_o)$.

**Data Structures:**
The model assumes a double-sided surface. Parameters $\beta_c, \beta_m, \beta_s$ are treated as scalar fields or constants per material instance. In the context of ABSDF, these are treated as random variables with joint probability density functions (PDFs) $p(\beta_c, \beta_m)$ and normal distribution $p_N(n)$ [S1, unofficial].

**Formulas:**
The total BRDF is the sum of diffuse and specular components:
$$ f_{disney}(\omega_i, \omega_o) = f_d(\omega_i, \omega_o) + f_s(\omega_i, \omega_o) $$

1.  **Diffuse Component ($f_d$):**
    Uses a Lambertian model scaled by the base color and a non-metallic factor:
    $$ f_d(\omega_i, \omega_o) = \frac{(1 - \beta_m)\beta_c}{\pi} $$
    [S1, unofficial]

2.  **Specular Component ($f_s$):**
    Combines metallic and dielectric Fresnel reflections using the microfacet model:
    $$ f_s(\omega_i, \omega_o) = \frac{D(\omega_h; \alpha)G(\omega_i, \omega_o; \alpha)}{4|n \cdot \omega_o||n \cdot \omega_i|} \left[ \beta_m F(\omega_h, \omega_o; \beta_c) + (1 - \beta_m)F(\omega_h, \omega_o; \beta_s) \right] $$
    [S1, unofficial]

    *   **Microfacet Distribution ($D$):** Trowbridge-Reitz (GGX) distribution $D(\omega_h; \alpha)$ [S1, unofficial].
    *   **Shadowing-Masking ($G$):** Smith masking-shadowing function $G(\omega_i, \omega_o; \alpha)$ [S1, unofficial].
    *   **Fresnel ($F$):** Schlick Fresnel approximation:
        $$ F(\omega_h, \omega_o; r_0) = r_0(1 - F_c) + F_c $$
        $$ F_c = (1 - |\omega_h \cdot \omega_o|)^5 $$
        where $r_0$ is the normal incidence reflectance. For the metallic lobe, $r_0 = \beta_c$. For the dielectric lobe, $r_0 = \beta_s$ [S1, unofficial].

**Steps:**
1.  Compute the half vector $\omega_h$.
2.  Evaluate the dot products $n \cdot \omega_i$, $n \cdot \omega_o$, and $\omega_h \cdot \omega_o$.
3.  Calculate the Schlick Fresnel term $F_c$ using the dot product $\omega_h \cdot \omega_o$.
4.  Calculate the Fresnel reflectance $F$ for both the metallic ($r_0 = \beta_c$) and dielectric ($r_0 = \beta_s$) cases.
5.  Evaluate the GGX distribution $D$ and Smith geometry function $G$ using roughness $\alpha$.
6.  Compute the specular term $f_s$ by blending the metallic and dielectric Fresnel terms weighted by $\beta_m$ and $(1-\beta_m)$.
7.  Compute the diffuse term $f_d$ using $\beta_c$ and $\beta_m$.
8.  Sum $f_d$ and $f_s$ to get the final BRDF.

**Edge Cases and Pitfalls:**
*   **Double-Sided Assumption:** The model assumes the surface is double-sided, meaning the BRDF is evaluated regardless of the sign of $n \cdot \omega_i$ or $n \cdot \omega_o$ in the aggregation context, though standard microfacet terms usually require valid visibility [S1, unofficial].
*   **Parameter Correlation:** The factorization assumes $\beta_c$ and $\beta_m$ are orientation-independent in the base case, but the method extends to orientation-varying parameters via moment-based approximations [S1, unofficial].

## Subtopics

### [Specular ABSDF Factorization](10872-specular-absdf-factorization.md)
Specular ABSDF Factorization is a method for aggregating the specular component of the ABSDF [10872-specular-absdf-factorization.md]. It solves the problem of efficiently rendering heterogeneous materials with varying roughness and orientation by factorizing the complex integral of the microfacet BRDF into separable terms involving Fresnel mix, microfacet distribution, and shadowing-masking. It is distinguished from its sibling, [Phys3DGS BRDF Function](11071-phys3dgs-brdf-function.md), by explicitly handling the statistical aggregation of roughness parameters via moment-based approximation rather than a direct physical simulation.

### [Phys3DGS BRDF Function](11071-phys3dgs-brdf-function.md)
A physically-based Bidirectional Reflectance Distribution Function (BRDF) used in 3D Gaussian Splatting for inverse rendering, consisting of a diffuse component and a specular component based on the GGX microfacet model with a correlated Smith masking-shadowing function. It differs from its parent, the Modified Disney Principled BRDF, by employing a specific roughness remapping ($\alpha = r^2$) and a correlated Smith geometry function rather than the separable Schlick-GGX approximation, aiming for higher physical accuracy in the context of Gaussian splatting optimization.

### Diffuse ABSDF Factorization
A method for factoring the diffuse component of the ABSDF [S1, unofficial]. It solves the problem of integrating the Lambertian diffuse term over spatially varying material parameters and surface normals. The diffuse integral is split into a parameter expectation term and a geometric term approximated by Spherical Gaussians (SG). The parameter integral is computed as $E[\beta_c] - E[\beta_m \beta_c]$, while the geometric integral $\int_{S^2} \langle n \cdot \omega_i \rangle \langle n \cdot \omega_o \rangle p_N(n) dn$ is approximated by fitting the clamped dot product product to a single SG with amplitude $c$ and concentration $\kappa$, reducing the problem to an SG-SGGX convolution [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
The base evaluation is $O(1)$ per sample. The ABSDF factorization reduces the complexity of rendering heterogeneous materials by replacing expensive Monte Carlo integration over parameter distributions with closed-form moment calculations and SG convolutions [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** High compatibility with existing production assets (Disney BRDF); enables efficient LOD rendering and aggregation of heterogeneous materials; simplified diffuse term allows for analytical integration [S1, unofficial].
*   **Limitations:** Omits sheen and clearcoat, limiting appearance range for certain materials (e.g., hair, glossy plastics); Lambertian diffuse is less accurate for non-Lambertian surfaces compared to the original Disney model [S1, unofficial].

**When to Use:**
Use when rendering heterogeneous materials in a Level-of-Detail (LOD) framework where efficient aggregation of spatially varying parameters is required [S1, unofficial].

**When Not to Use:**
Do not use if high-fidelity sheen or clearcoat effects are required, or if the surface exhibits strong non-Lambertian diffuse behavior that cannot be approximated by a simple albedo scaling [S1, unofficial].

## Background
The Modified Disney Principled BRDF originates from the work on "Efficient Scene Appearance Aggregation for Level-of-Detail Rendering" [S1, unofficial]. It was developed to support the widest applicability for ABSDF by targeting the Disney Principled BRDF [Burley 2012], one of the most common models in production. The modifications (Lambertian diffuse, no sheen/clearcoat, double-sided) were made to facilitate the mathematical factorization required for efficient aggregation [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
