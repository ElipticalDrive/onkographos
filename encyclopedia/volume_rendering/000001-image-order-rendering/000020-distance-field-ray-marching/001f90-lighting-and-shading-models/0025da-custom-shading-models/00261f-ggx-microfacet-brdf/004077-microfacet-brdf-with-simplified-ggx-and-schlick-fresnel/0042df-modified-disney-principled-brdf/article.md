# Modified Disney Principled BRDF

Also called: ABSDF Base Material, Base Material Model

## Overview
A simplified material model based on the Disney Principled BRDF, modified specifically to enable efficient aggregation in the Aggregated Bidirectional Surface Distribution Function (ABSDF) framework [S1, unofficial]. It solves the problem of rendering complex, orientation-varying microfacet materials in large-scale scenes by reducing the original model's complexity to a form that supports analytical factorization of diffuse and specular components [S1, unofficial]. Unlike the full Disney model, it omits sheen and clearcoat lobes and uses a Lambertian diffuse term, enabling real-time performance via precomputed tables and separable integrals [S1, unofficial].

## How it works
**Inputs:**
- Incoming light direction $\omega_i$.
- Outgoing view direction $\omega_o$.
- Surface normal $n$.
- Material parameters: roughness $\alpha$, base color $\beta_c$, metallic $\beta_m$, specular intensity $\beta_s$. These parameters can be spatially varying [S1, unofficial].

**Outputs:**
- Radiance $L$ or reflectance $f_r(\omega_i, \omega_o)$.

**Data Structures:**
- Material parameter tuple $\beta_B = (\alpha, \beta_c, \beta_m, \beta_s)$ [S1, unofficial].
- Half vector $\omega_h = \frac{\omega_i + \omega_o}{\|\omega_i + \omega_o\|}$ [S1, unofficial].

**Steps:**
1. Compute the half vector $\omega_h$ from $\omega_i$ and $\omega_o$.
2. Calculate the diffuse component $f_d$ using the Lambertian model scaled by material parameters.
3. Calculate the specular component $f_s$ using the microfacet model with GGX distribution, Schlick Fresnel, and Smith shadowing-masking.
4. Sum the components: $f_{disney} = f_d + f_s$.

**Formulas:**
The total reflectance is:
$$ f_{disney}(\omega_i, \omega_o) = f_d(\omega_i, \omega_o) + f_s(\omega_i, \omega_o) $$

The diffuse component is:
$$ f_d(\omega_i, \omega_o) = \frac{(1 - \beta_m)\beta_c}{\pi} $$
where $\beta_c$ is the base color and $\beta_m$ is the metallic parameter [S1, unofficial].

The specular component is:
$$ f_s(\omega_i, \omega_o) = \frac{D(\omega_h; \alpha)G(\omega_i, \omega_o; \alpha)}{4|n \cdot \omega_o||n \cdot \omega_i|} \left( \beta_m F(\omega_h, \omega_o; \beta_c) + (1 - \beta_m)F(\omega_h, \omega_o; \beta_s) \right) $$
where:
- $D(\omega_h; \alpha)$ is the Trowbridge-Reitz (GGX) Normal Distribution Function (NDF) [S1, unofficial].
- $G(\omega_i, \omega_o; \alpha)$ is the shadowing-masking function [S1, unofficial].
- $F(\omega_h, \omega_o; r_0)$ is the Schlick Fresnel reflectance [S1, unofficial].

The Schlick Fresnel function is defined as:
$$ F(\omega_h, \omega_o; r_0) = r_0 (1 - F_c) + F_c $$
$$ F_c = (1 - |\omega_h \cdot \omega_o|)^5 $$
where $r_0$ is the normal incidence reflectance. For the metallic lobe, $r_0 = \beta_c$. For the dielectric (non-metallic) lobe, $r_0 = \beta_s$ [S1, unofficial].

**Parameters:**
- $\alpha$: Roughness.
- $\beta_c$: Base color (used for diffuse albedo and metallic $r_0$).
- $\beta_m$: Metallic (0 to 1).
- $\beta_s$: Specular intensity (dielectric $r_0$) [S1, unofficial].

**Edge Cases and Pitfalls:**
- The model assumes double-sided surfaces [S1, unofficial].
- The diffuse term uses a simple Lambertian model, omitting the original Disney empirical retro-reflection term [S1, unofficial].
- Sheen and clearcoat lobes are omitted [S1, unofficial].

## Subtopics

### [Specular ABSDF Factorization](10872-specular-absdf-factorization.md)
Specular ABSDF Factorization is the specular component of the Aggregated Bidirectional Surface Distribution Function (ABSDF) [10860-modified-disney-principled-brdf.md], designed to efficiently render complex, orientation-varying microfacet materials in large-scale scenes. It solves the problem of integrating the specular BRDF over a distribution of surface orientations and material parameters by factorizing the integral into separable terms: a precomputable convolution of the Normal Distribution Function (NDF) and a runtime-evaluated moment term. Unlike [Phys3DGS BRDF Function](11071-phys3dgs-brdf-function.md), which may rely on different aggregation strategies, this method explicitly handles the coupling between Fresnel terms, roughness, and microfacet geometry through a split of the integral into "clear coat" and "base" contributions, allowing for real-time performance via precomputed tables.

### [Phys3DGS BRDF Function](11071-phys3dgs-brdf-function.md)
A physically-based BRDF function consisting of a diffuse and a specular component, using roughness remapping and a correlated Smith masking-shadowing function.

### Diffuse ABSDF Factorization
The diffuse component of the ABSDF factorization. It assumes $\beta_c$ and $\beta_m$ are orientation-independent initially, splitting the integral into a parameter expectation term and a geometric term. The parameter integral is represented by means and second-order moments: $\int_{[0,1]^2} (1 - \beta_m)\beta_c p(\beta_c, \beta_m) d\beta_c d\beta_m = E[\beta_c] - E[\beta_m \beta_c]$ [S1, unofficial]. The geometric integral $\int_{S^2} \langle n \cdot \omega_i \rangle \langle n \cdot \omega_o \rangle p_N(n) dn$ is approximated by fitting the clamped dot product function with a Spherical Gaussian (SG), reducing the problem to the convolution between an SG and an SGGX [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Depends on the evaluation of GGX NDF, Smith G, and Schlick Fresnel. For ABSDF, precomputed tables and convolutions allow for efficient runtime evaluation [S1, unofficial].
- **Memory Complexity:** Requires storage for material parameters $\beta_B$ and potentially precomputed tables for ABSDF factorization [S1, unofficial].
- **Strengths:** Simplified from the full Disney model, enabling analytical factorization for large-scale scene aggregation. Supports spatially varying parameters [S1, unofficial].
- **Limitations:** Omits sheen and clearcoat, limiting appearance range compared to the full Disney model. Lambertian diffuse may not capture complex subsurface or retro-reflection effects [S1, unofficial].
- **When to use:** When rendering large-scale scenes with orientation-varying materials where performance is critical and the full Disney model's complexity is unnecessary or prohibitive for aggregation [S1, unofficial].

## Background
Originated in the context of the Aggregated Bidirectional Surface Distribution Function (ABSDF) for level-of-detail rendering [S1, unofficial]. It is based on the Disney Principled BRDF [Burley 2012], which is one of the most commonly used models in production [S1, unofficial]. The modifications were made to preserve the core features of the Disney model while enabling efficient factorization for the ABSDF framework [S1, unofficial].

## Sources
[S1] Efficient Scene Appearance Aggregation for Level-of-Detail Rendering, https://core.ac.uk/works/391729893, fetched 2026-11-01 (unofficial)
