# GGX Microfacet BRDF

Also called: GGX BRDF, GGX Microfacet BRDF Model, GGX Microfacet Model, GGX Microfacet Specular Model, GGX Shading, Walter et al. Specular

## Overview
The GGX Microfacet BRDF models specular reflection by assuming the surface consists of microscopic facets with normals distributed according to the Generalized Trowbridge-Reiss (GGX) distribution [S1, unofficial]. It serves as a physically-based material model for realistic metal and plastic surfaces [S3]. It builds on [Custom Shading Models](3660-custom-shading-models.md) by decomposing surface reflection into diffuse and specular microfacet components parameterized by diffuse albedo $K_d$, specular albedo $K_s$, and roughness $\rho$ [S4].

## How it works
**Inputs and Outputs:**
*   Surface normal $\mathbf{n}$ (unit vector) [G].
*   View direction $\mathbf{v}$ (unit vector, from surface to camera) [G].
*   Light direction $\mathbf{l}$ (unit vector, from surface to light) [G].
*   Roughness $\rho$ (scalar, typically $[0, 1]$) [S1, unofficial][S4].
*   Diffuse albedo $K_d$ (RGB vector) [S4].
*   Specular albedo $K_s$ (RGB vector, often derived from $F_0$ for metals) [S4].
*   Outputs: Diffuse BRDF $f_d$, specular BRDF $f_s$, and total reflected radiance $L = f_d + f_s$ [S4].

**Data Structures:**
No complex data structures are required. The computation relies on scalar and vector arithmetic [G].

**Parameters:**
*   $\rho$: Roughness. Typical values range from $0.0$ (smooth) to $1.0$ (rough) [S1, unofficial][S4].
*   $F_0$: Specular reflectance at normal incidence. A typical value is $0.14$ [S1, unofficial].
*   $K_d$: Diffuse albedo [S4].
*   $K_s$: Specular albedo at normal incidence [S4].

**Steps and Formulas:**
1.  **Compute Half Vector:**
    $$ \mathbf{h} = \frac{\mathbf{v} + \mathbf{l}}{\|\mathbf{v} + \mathbf{l}\|} $$
    [G] If $\mathbf{v} \approx -\mathbf{l}$, $\mathbf{h}$ is undefined; use $\mathbf{n}$ or a default reflection vector [G].

2.  **Compute Dot Products:**
    $$ \mu_v = \max(\mathbf{n} \cdot \mathbf{v}, 0.0) $$
    $$ \mu_l = \max(\mathbf{n} \cdot \mathbf{l}, 0.0) $$
    $$ \mu_h = \max(\mathbf{n} \cdot \mathbf{h}, 0.0) $$
    $$ \mu_{lh} = \max(\mathbf{l} \cdot \mathbf{h}, 0.0) $$
    [G]

3.  **Compute Alpha:**
    $$ \alpha = \rho^2 $$
    [S1, unofficial][S4]

4.  **Compute Normal Distribution Function (D):**
    The GGX distribution describes the probability density of microfacet normals [S2, unofficial].
    $$ D(\alpha, \mathbf{n}, \mathbf{h}) = \frac{\alpha^2}{\pi \left( (\alpha^2 - 1)\mu_h^2 + 1 \right)^2} $$
    [S2, unofficial][S4]

5.  **Compute Geometry Function (G):**
    The Smith geometry function is standard [S2, unofficial].
    $$ k = \frac{(\rho + 1)^2}{8} $$
    [S2, unofficial]
    $$ G_1(\mathbf{n}, \mathbf{v}, \alpha) = \frac{\mu_v}{\max(\mu_v(1 - k) + k, \epsilon)} $$
    [S2, unofficial]
    $$ G_2(\alpha, \mathbf{n}, \mathbf{v}, \mathbf{l}) = G_1(\mathbf{n}, \mathbf{v}, \alpha) \cdot G_1(\mathbf{n}, \mathbf{l}, \alpha) $$
    [S2, unofficial]

6.  **Compute Fresnel Function (F):**
    The Schlick approximation is standard [S1, unofficial].
    $$ F(F_0, \mu_{lh}) = F_0 + (1 - F_0)(1 - \mu_{lh})^5 $$
    [S1, unofficial]
    For variable specular albedo $K_s$, the reflectance at grazing angles is computed as:
    $$ F_{90}(K_s) = \min\left(\frac{\text{lum}(K_s)}{0.04}, 1.0\right) $$
    [S4]
    $$ \text{lum}(C) = 0.213 C_r + 0.715 C_g + 0.072 C_b $$
    [S4]
    $$ F(K_s, \mu_{lh}) = K_s + (F_{90}(K_s) - K_s)(1 - \mu_{lh})^5 $$
    [S4]

7.  **Compute Specular BRDF:**
    $$ f_s(\mathbf{v}, \mathbf{l}) = \frac{D(\alpha, \mathbf{n}, \mathbf{h}) G_2(\alpha, \mathbf{n}, \mathbf{v}, \mathbf{l}) F(K_s, \mu_{lh})}{4 \mu_v \mu_l} $$
    [S4]

8.  **Compute Diffuse BRDF:**
    $$ f_d(K_d) = \frac{K_d}{\pi} $$
    [S4]

9.  **Combine:**
    $$ f_r(\mathbf{v}, \mathbf{l}) = f_d(K_d) + f_s(\mathbf{v}, \mathbf{l}) $$
    [S4]

**Edge Cases and Pitfalls:**
Epsilon values (e.g., $10^{-6}$) are added to the denominator of $D$ and the specular term to prevent division by zero [S1, unofficial][S2, unofficial]. The denominator $4 \mu_v \mu_l$ requires a guard (e.g., $10^{-4}$) when the view or light direction is nearly parallel to the surface [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def ggx_microfacet_brdf(n, v, l, roughness, kd, ks):
    # 1. Half vector
    h = v + l
    h_len = math.sqrt(h[0]**2 + h[1]**2 + h[2]**2)
    if h_len < 1e-6:
        h = n  # Fallback for antiparallel v and l
    else:
        h = (h[0]/h_len, h[1]/h_len, h[2]/h_len)
    
    # 2. Dot products
    nv = max(n[0]*v[0] + n[1]*v[1] + n[2]*v[2], 0.0)
    nl = max(n[0]*l[0] + n[1]*l[1] + n[2]*l[2], 0.0)
    nh = max(n[0]*h[0] + n[1]*h[1] + n[2]*h[2], 0.0)
    lh = max(l[0]*h[0] + l[1]*h[1] + l[2]*h[2], 0.0)
    
    # 3. Alpha
    alpha = roughness * roughness
    alpha2 = alpha * alpha
    
    # 4. D (GGX)
    denom = (alpha2 - 1.0) * (nh * nh) + 1.0
    D = alpha2 / (math.pi * denom * denom + 1e-6)
    
    # 5. G (Smith Geometry)
    k = ((roughness + 1.0) ** 2) / 8.0
    g1_v = nv / max(nv * (1.0 - k) + k, 1e-6)
    g1_l = nl / max(nl * (1.0 - k) + k, 1e-6)
    G = g1_v * g1_l
    
    # 6. F (Schlick)
    lum_ks = 0.213 * ks[0] + 0.715 * ks[1] + 0.072 * ks[2]
    f90 = min(lum_ks / 0.04, 1.0)
    F = ks + (f90 - ks) * ((1.0 - lh) ** 5)
    
    # 7. Specular BRDF
    specular = (D * G * F) / max(4.0 * nv * nl, 1e-4)
    
    # 8. Diffuse BRDF
    diffuse = kd / math.pi
    
    # 9. Total
    return diffuse + specular
```

## Subtopics

### [Specular Reflection](5690-specular-reflection.md)
Specular reflection models perfectly smooth surfaces where outgoing radiance is not scattered but mirrors the incident trajectory across the surface normal. It serves as the zero-roughness limit of microfacet models, producing perfect mirror reflections. Unlike GGX or Ward models which distribute energy over a lobe, this method concentrates all reflected energy into a single geometric direction, requiring exact alignment between the incident light and the view vector relative to the surface normal.

### [Microfacet BRDF with Simplified GGX and Schlick Fresnel](10259-microfacet-brdf-with-simplified-ggx-and-schlick-fresnel.md)
This method defines a surface reflectance function $R(\mathbf{x}, \boldsymbol{\omega}_i, \boldsymbol{\omega}_o)$ that combines a specular microfacet term with a diffuse Lambertian term, incorporating specific simplifications from the Filament and Unreal Engine renderers. It sets itself apart from the parent [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) by using a specific analytical form for the geometry function $G$ and embedding the incoming cosine term $(\mathbf{n} \cdot \boldsymbol{\omega}_i)$ directly into the specular denominator structure to simplify integration in neural radiance field contexts.

### [Ward Anisotropic BRDF](19058-ward-anisotropic-brdf.md)
The Ward Anisotropic BRDF is a microfacet specular reflection model that modifies the Beckmann distribution to simulate surface roughness varying in two orthogonal directions relative to the tangent space. It is distinguished from its siblings by its specific parameterization of anisotropy via the ratio of eigenvalues from a covariance matrix, allowing for position-dependent control of highlight stretching. Unlike the isotropic GGX model, it explicitly decouples roughness along the tangent and bitangent axes.

### [GGX Specular Shading](21197-ggx-specular-shading.md)
GGX Specular Shading is a path-tracing shading method for voxel-based volume rendering that combines a Cook-Torrance microfacet model with GGX specular reflection, Lambertian diffuse shading from an irradiance cache, and dielectric transmission handling. It differs from its parent [GGX Microfacet BRDF](3715-ggx-microfacet-brdf.md) by providing the complete shading loop (emission, reflection, refraction, and diffuse accumulation) rather than just the BRDF components. It is distinct from [Specular Reflection](5690-specular-reflection.md) by including transmission and diffuse GI, and from [Microfacet BRDF with Simplified GGX and Schlick Fresnel](10259-microfacet-brdf-with-simplified-ggx-and-schlick-fresnel.md) by using the full Cook-Torrance formulation with explicit transmission logic.

### Multi-Scattering Energy Compensation for Rough Metals
A method to ensure energy conservation for rough metals by compensating for energy lost in single-scattering GGX specular lobes, using a precomputed directional albedo table [S5, unofficial]. It uses a precomputed 32x32 directional albedo table $E(\mu, \alpha)$, where $\mu$ is cosine and $\alpha$ is roughness [S5, unofficial]. This ensures white rough metal reflects ~100% at every roughness, correcting the energy loss inherent in single-scattering approximations [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [G] Standard microfacet evaluation is $O(1)$ per shading point.
*   **Memory Complexity:** Not given in the sources. [G] Base model requires $O(1)$ memory; multi-scattering variants require a 32x32 lookup table [S5, unofficial].
*   **Strengths and Limitations:** Not given in the sources. [G] Handles a wide range of roughnesses but single-scattering approximation loses energy for very rough metals, addressed by multi-scattering variants [S5, unofficial]. Does not account for subsurface scattering or thin-film effects without extensions.
*   **When to use:** Not given in the sources. [G] General-purpose physically based rendering for rough surfaces.
*   **When not to use:** Not given in the sources. [G] Perfect mirrors or when computational budget is extremely tight.

## Background
The GGX microfacet model was introduced by Walter et al. in "Microfacet Models for Refraction through Rough Surfaces" [S1, unofficial]. It uses the Generalized Trowbridge-Reiss (GGX) distribution to approximate surface reflection properties, providing a better fit for the long tails of specular highlights observed in real-world materials compared to earlier distributions. It is referenced in implementations for realistic metal and plastic surfaces [S3].

## Sources
[S1] GitHub - razzle-reinizu/Voxel-World-Shader: A GLSL voxel-world shader ..., https://github.com/razzle-reinizu/Voxel-World-Shader, fetched 2026-10-25 (unofficial)
[S2] TeoGrajqevci/RayMarching_tool, https://github.com/TeoGrajqevci/RayMarching_tool, fetched 2026-10-26 (unofficial)
[S3] MagicaVoxel .vox, https://ephtracy.github.io/, fetched 2026-10-26
[S4] I 2 -SDF: Intrinsic Indoor Scene Reconstruction and Editing via Raytracing in Neural SDFs, https://doi.org/10.1109/cvpr52729.2023.01202, fetched 2026-11-01
[S5] Add volumetric rendering, advanced materials, and imager effects, https://github.com/MihrimatriX/photon-engine/pull/3, fetched 2026-10-28 (unofficial)
