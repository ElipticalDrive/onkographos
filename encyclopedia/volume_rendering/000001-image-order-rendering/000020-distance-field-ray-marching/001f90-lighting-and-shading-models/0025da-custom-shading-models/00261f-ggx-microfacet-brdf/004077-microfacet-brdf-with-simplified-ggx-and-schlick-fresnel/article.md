# Microfacet BRDF with Simplified GGX and Schlick Fresnel

Also called: NeRV BRDF, Walter et al. BRDF with Filament/Unreal simplifications

## Overview
This method defines a microfacet-based Bidirectional Reflectance Distribution Function (BRDF) that combines the GGX normal distribution with Schlick-GGX geometry and Fresnel approximations, as implemented in the Filament and Unreal Engine renderers [S1]. It is distinguished from the parent [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) by its specific algebraic simplifications for the distribution and geometry terms, and by the inclusion of a Lambertian diffuse term multiplied by the cosine of the incident angle within the BRDF definition itself [S1]. This formulation is primarily used in neural radiance field variants like NeRV to enable efficient factorization of light transport [S1].

## How it works
**Inputs:**
- $\mathbf{x}$: 3D surface location.
- $\boldsymbol{\omega}_i$: Incoming light direction (unit vector).
- $\boldsymbol{\omega}_o$: Outgoing view direction (unit vector).
- $\mathbf{n}$: Surface normal at $\mathbf{x}$ (unit vector).
- $\gamma$: Roughness parameter ($\gamma > 0$).
- $\mathbf{a}$: Diffuse albedo (scalar or RGB vector).
- $F_0$: Fresnel reflectance at normal incidence (scalar, typically 0.04 for dielectrics) [S1].

**Outputs:**
- $R(\mathbf{x}, \boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$: Reflectance value.

**Data Structures:**
No complex data structures are required; the computation relies on standard vector operations and scalar parameters.

**Steps:**
1.  Compute the half-vector $\mathbf{h}$:
    $$ \mathbf{h} = \frac{\boldsymbol{\omega}_o + \boldsymbol{\omega}_i}{\|\boldsymbol{\omega}_o + \boldsymbol{\omega}_i\|} $$
2.  Compute intermediate scalar $\rho$ and geometry parameter $k$:
    $$ \rho = \gamma^2 $$
    $$ k = \frac{\gamma^4}{2} $$
3.  Compute dot products:
    $$ n_h = \mathbf{n} \cdot \mathbf{h} $$
    $$ n_{\omega_i} = \mathbf{n} \cdot \boldsymbol{\omega}_i $$
    $$ n_{\omega_o} = \mathbf{n} \cdot \boldsymbol{\omega}_o $$
    $$ \omega_i_h = \boldsymbol{\omega}_i \cdot \mathbf{h} $$
4.  Compute the Normal Distribution Function (NDF) $D$:
    $$ D(\mathbf{h}, \mathbf{n}, \gamma) = \frac{\rho^2}{\pi \left( n_h^2 (\rho^2 - 1) + 1 \right)^2} $$
5.  Compute the Fresnel term $F$ using Schlick's approximation:
    $$ F(\boldsymbol{\omega}_i, \mathbf{h}) = F_0 + (1 - F_0)(1 - \omega_i_h)^5 $$
6.  Compute the Geometry term $G$ (Smith approximation):
    $$ G(\boldsymbol{\omega}_i, \boldsymbol{\omega}_o, \gamma) = \frac{n_{\omega_o} n_{\omega_i}}{ \left( n_{\omega_o}(1 - k) + k \right) \left( n_{\omega_i}(1 - k) + k \right) } $$
7.  Compute the specular reflectance component:
    $$ R_{spec} = \frac{D \cdot F \cdot G}{4 n_{\omega_o}} $$
8.  Compute the diffuse reflectance component. Note that this term includes the Lambert cosine factor $(\mathbf{n} \cdot \boldsymbol{\omega}_i)$ as part of the BRDF definition to simplify integral equations [S1]:
    $$ R_{diff} = n_{\omega_i} (1 - F(\boldsymbol{\omega}_i, \mathbf{h})) \frac{\mathbf{a}}{\pi} $$
9.  Combine components:
    $$ R(\mathbf{x}, \boldsymbol{\omega}_i, \boldsymbol{\omega}_o) = R_{spec} + R_{diff} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_simplified_ggx_brdf(n, wi, wo, gamma, a, F0):
    # 1. Half vector
    h = normalize(wo + wi)
    
    # 2. Parameters
    rho = gamma * gamma
    k = (gamma ** 4) / 2.0
    
    # 3. Dot products
    n_dot_h = dot(n, h)
    n_dot_wi = dot(n, wi)
    n_dot_wo = dot(n, wo)
    wi_dot_h = dot(wi, h)
    
    # 4. D (GGX Normal Distribution)
    # D = rho^2 / (pi * ((n_dot_h^2 * (rho^2 - 1) + 1)^2))
    denom_d = (n_dot_h ** 2) * (rho ** 2 - 1) + 1.0
    D = (rho ** 2) / (pi * (denom_d ** 2))
    
    # 5. F (Schlick Fresnel)
    # F = F0 + (1 - F0) * (1 - wi_dot_h)^5
    F = F0 + (1.0 - F0) * ((1.0 - wi_dot_h) ** 5)
    
    # 6. G (Smith Geometry)
    # G = (n_dot_wo * n_dot_wi) / ((n_dot_wo * (1 - k) + k) * (n_dot_wi * (1 - k) + k))
    denom_g1 = n_dot_wo * (1.0 - k) + k
    denom_g2 = n_dot_wi * (1.0 - k) + k
    G = (n_dot_wo * n_dot_wi) / (denom_g1 * denom_g2)
    
    # 7. Specular term
    # R_spec = (D * F * G) / (4 * n_dot_wo)
    # Guard against division by zero if n_dot_wo is near zero
    if n_dot_wo < 1e-6:
        R_spec = 0.0
    else:
        R_spec = (D * F * G) / (4.0 * n_dot_wo)
        
    # 8. Diffuse term
    # R_diff = n_dot_wi * (1 - F) * (a / pi)
    # Note: The source [S1] explicitly includes n_dot_wi in the BRDF definition
    R_diff = n_dot_wi * (1.0 - F) * (a / pi)
    
    # 9. Total
    return R_spec + R_diff
```

**Parameters:**
- $F_0$: Set to 0.04 for dielectric materials [S1].
- $\gamma$: Roughness parameter [S1].
- $\mathbf{a}$: Diffuse albedo [S1].

**Edge Cases and Pitfalls:**
- The BRDF definition includes the multiplication by the Lambert cosine term $(\mathbf{n} \cdot \boldsymbol{\omega}_i)$ in the diffuse term [S1]. This means the geometric term $G$ and the diffuse term are structured differently than standard separable forms, which must be accounted for when integrating or factorizing the BRDF [S1].
- Division by zero in the specular term if $\mathbf{n} \cdot \boldsymbol{\omega}_o$ is zero or negative (grazing/back-face). A guard is required [G].

## Subtopics

### [Modified Disney Principled BRDF](10860-modified-disney-principled-brdf.md)
A simplified material model based on the Disney Principled BRDF [Burley 2012], modified specifically to enable efficient Aggregated Bidirectional Scattering Distribution Function (ABSDF) factorization. It solves the problem of rendering heterogeneous materials with varying roughness and orientation by stripping the original model down to its core diffuse and specular components, allowing for separable integral approximations. It is distinguished from its parent, [Microfacet BRDF with Simplified GGX and Schlick Fresnel](10259-microfacet-brdf-with-simplified-ggx-and-schlick-fresnel.md), by explicitly adopting a Lambertian diffuse term and omitting sheen/clearcoat lobes to facilitate the statistical aggregation required for ABSDF.

## Operating characteristics
Not given in the sources.

## Background
This BRDF formulation is based on the standard microfacet theory described by Walter et al. [55] and incorporates simplifications from the Filament [15] and Unreal Engine [20] rendering engines [S1]. It was notably used in the NeRV (Neural Reflectance and Visibility Fields) method for relighting and view synthesis [S1].

## Sources
[S1] NeRV: Neural Reflectance and Visibility Fields for Relighting and View Synthesis, https://doi.org/10.1109/cvpr46437.2021.00741, fetched 2026-10-26
