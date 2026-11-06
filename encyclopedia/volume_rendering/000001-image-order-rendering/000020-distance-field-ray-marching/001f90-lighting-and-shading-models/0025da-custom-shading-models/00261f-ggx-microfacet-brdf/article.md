# GGX Microfacet BRDF

Also called: GGX BRDF, GGX Microfacet Model, GGX Microfacet Specular Model, GGX Shading, Walter et al. Specular

## Overview
The GGX Microfacet BRDF is a specular reflection model that approximates surface scattering using a Gaussian distribution of microfacet normals (GGX) combined with the Schlick Fresnel approximation and a Smith geometry function [S2, unofficial]. It provides a physically plausible long-tail specular response for rough surfaces [S1, unofficial].

## How it works
**Inputs:** Surface normal $N$, view direction $V$, light direction $L$, roughness parameter $\alpha_{rough}$, base specular reflectance $F_0$ [G].
**Outputs:** Specular reflection coefficient $f_r$ [G].

**Data Structures:**
No complex data structures are required for the standard implementation; inputs are scalar or vector values [G].

**Steps:**
1.  Compute the half-vector $H = \text{normalize}(L + V)$ [G]. (Note: Source [S1, unofficial] uses $L - V$ in its implementation [S1, unofficial].)
2.  Compute squared roughness $\alpha = \alpha_{rough}^2$ [S1, S2, unofficial].
3.  Compute the GGX Normal Distribution Function (NDF) $D(N, H, \alpha)$ [S1, S2, unofficial]:
    $$ D = \frac{\alpha^2}{\pi \left( (N \cdot H)^2 (\alpha^2 - 1) + 1 \right)^2 + \epsilon_D} $$
    where $N \cdot H = \max(\text{dot}(N, H), 0.0)$ and $\epsilon_D = 10^{-6}$ prevents division by zero [S1, S2, unofficial].
4.  Compute the geometry parameter $k$. Source [S2, unofficial] defines $k = (\alpha_{rough} + 1)^2 / 8$ for the standard Smith formulation, while Source [S1, unofficial] uses the approximation $k = 0.5 \alpha$ [S1, S2, unofficial].
5.  Compute the single-scattering geometry term $G_1$ for view and light directions [S2, unofficial]:
    $$ G_1(V) = \frac{N \cdot V}{\max((N \cdot V)(1 - k) + k, \epsilon_G)} $$
    $$ G_1(L) = \frac{N \cdot L}{\max((N \cdot L)(1 - k) + k, \epsilon_G)} $$
    where $N \cdot V = \max(\text{dot}(N, V), 0.0)$, $N \cdot L = \max(\text{dot}(N, L), 0.0)$, and $\epsilon_G = 10^{-6}$ [S2, unofficial].
6.  Compute the total geometry term $G = G_1(V) \cdot G_1(L)$ [S2, unofficial].
7.  Compute the Schlick Fresnel term $F(L, H, F_0)$ [S1, S2, unofficial]:
    $$ F = F_0 + (1 - F_0)(1 - L \cdot H)^5 $$
    where $L \cdot H = \max(\text{dot}(L, H), 0.0)$ [S1, S2, unofficial].
8.  Compute the final specular BRDF value [S1, S2, unofficial]:
    $$ f_r = \frac{D \cdot G \cdot F}{4 (N \cdot V) (N \cdot L) + \epsilon_{denom}} $$
    where $\epsilon_{denom} = 10^{-4}$ [S2, unofficial]. (Source [S1, unofficial] uses a compact alternative denominator $\max(10^{-6}, (L \cdot H)^2 (1 - k^2)) + k^2$ [S1, unofficial].)

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def normalize(v):
    l = math.sqrt(dot(v, v))
    return (v[0]/l, v[1]/l, v[2]/l) if l > 0 else (0.0, 0.0, 0.0)

def ggx_microfacet_brdf(N, V, L, roughness, F0):
    PI = math.pi
    # 1. Half vector
    H = normalize(L + V)
    
    # 2. Squared roughness
    alpha = roughness * roughness
    alpha_sqr = alpha * alpha
    
    # 3. GGX NDF
    dot_NH = max(dot(N, H), 0.0)
    denom_D = dot_NH * dot_NH * (alpha_sqr - 1.0) + 1.0
    D = alpha_sqr / (PI * denom_D * denom_D + 1e-6)
    
    # 4. Geometry parameter k (Standard Smith per [S2, unofficial])
    k = (roughness + 1.0) ** 2 / 8.0
    
    # 5. Geometry terms
    dot_NV = max(dot(N, V), 0.0)
    dot_NL = max(dot(N, L), 0.0)
    
    G1_V = dot_NV / max(dot_NV * (1.0 - k) + k, 1e-6)
    G1_L = dot_NL / max(dot_NL * (1.0 - k) + k, 1e-6)
    G = G1_V * G1_L
    
    # 6. Fresnel
    dot_LH = max(dot(L, H), 0.0)
    F = F0 + (1.0 - F0) * (1.0 - dot_LH) ** 5
    
    # 7. Final BRDF
    denom_spec = 4.0 * dot_NV * dot_NL + 1e-4
    specular = (D * G * F) / denom_spec
    
    return specular
```

**Parameters:**
*   `roughness`: Surface roughness. Typical value: 0.54 [S1, unofficial].
*   `F0`: Base specular reflectance. Typical value: 0.14 for dielectrics [S1, unofficial]. For metals, this is derived from measured refractive index data [S3, unofficial].

**Edge Cases:**
*   Division by zero is prevented by adding epsilon values ($10^{-6}$ or $10^{-4}$) to denominators in $D$ and the final specular term [S1, S2, unofficial].
*   Degenerate inputs (e.g., $N \cdot V = 0$) result in zero specular contribution due to the geometry term or denominator guards [G].

## Subtopics

### Multi-Scattering Energy Compensation for Rough Metals
A method to ensure energy conservation for rough metals by compensating for energy lost in single-scattering GGX specular lobes [S3, unofficial]. It uses a precomputed 32x32 directional albedo table $E(\mu, \alpha)$, where $\mu$ is cosine and $\alpha$ is roughness [S3, unofficial]. This ensures that white rough metal reflects ~100% at every roughness, correcting the energy loss inherent in single-scattering approximations [S3, unofficial].

### Specular Reflection
Models perfectly smooth surfaces where outgoing radiance is not scattered but mirrors the incident trajectory across the surface normal [S4, unofficial]. The BRDF is modeled with a Dirac delta function: $f_{r,m} = \frac{\delta(\cos \theta_i - \cos \theta_r)}{\cos \theta_i} \delta(\phi_i - (\phi_r \pm \pi))$ [S4, unofficial]. The Dirac delta function yields zero for any non-zero parameter mismatch, meaning no light is reflected if the geometric reflection conditions are not perfectly satisfied [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per light per surface point [G].
*   **Memory Complexity:** $O(1)$ for standard GGX. $O(1)$ lookup for multi-scattering compensation if the table is baked into shader memory [S3, unofficial].
*   **Performance:** Fast enough for real-time rendering. The Schlick approximation is computationally cheap compared to full Fresnel equations [S1, unofficial].
*   **Strengths:** Physically plausible highlights, handles rough surfaces well, widely supported in PBR pipelines [S1, S2, unofficial].
*   **Limitations:** Single-scattering approximation loses energy for very rough metals, requiring compensation [S3, unofficial].
*   **When to use:** Real-time rendering of rough surfaces where visual fidelity is important [G].
*   **When not to use:** When exact energy conservation for rough metals is critical without post-processing compensation, or when subsurface scattering is required [G].

## Background
The GGX distribution was introduced by Walter et al. in "Microfacet Models for Refraction through Rough Surfaces" (2007) [S1, unofficial]. It is based on the Trowbridge-Reitz distribution, which provides a better fit for many real-world materials compared to the Beckmann distribution [S1, unofficial]. The Schlick Fresnel approximation is a standard simplification used in real-time graphics [S1, unofficial]. The model is widely used in modern PBR engines and voxel shaders [S1, S2, unofficial].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] TeoGrajqevci/RayMarching_tool, https://github.com/TeoGrajqevci/RayMarching_tool, fetched 2026-10-26 (unofficial)
[S3] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
[S4] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
