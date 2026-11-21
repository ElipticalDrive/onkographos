# Isotropic Disney BRDF

Also called: Cook-Torrance based Disney BRDF, Disney BRDF Material Model, Simplified Disney BRDF

## Overview
The Isotropic Disney BRDF is a simplified material model based on the Cook-Torrance microfacet framework, designed to model reflectance for both metals and plastics in a unified manner [S1, unofficial]. It is a restricted subset of the full Disney Principled BRDF, optimized for differentiable rendering and optimization tasks where ambiguity from high-dimensional parameter spaces must be reduced [S2]. Unlike the full model, it restricts optimization to opaque surfaces with spatially varying diffuse albedo and global constants for roughness, specular albedo, and metalness [S1, unofficial][S2].

## How it works
**Inputs and Outputs:**
*   **Input:** Surface position $\mathbf{x}$, incoming light direction $\omega_i$, outgoing view direction $\omega_o$, normal $\mathbf{n}$.
*   **Parameters:**
    *   $a(\mathbf{x}) \in [0, 1]^3$: Diffuse albedo, spatially varying [S1, unofficial].
    *   $s \in [0, 1]$: Specular albedo, global constant [S1, unofficial].
    *   $\beta \in [0, 1]$: Surface roughness, global constant [S1, unofficial].
    *   $m \in [0, 1]$: Metalness, global constant [S1, unofficial].
*   **Output:** Reflected radiance $L_r(\omega_o)$.

**Data Structures:**
The model assumes a single material instance per surface or texture-mapped parameters. The albedo $a$ is stored as a 3-channel texture or array indexed by $\mathbf{x}$. The parameters $s, \beta, m$ are scalar constants or global uniforms.

**Formulas:**
The BRDF $f_r(\omega_i, \omega_o)$ is composed of a diffuse term and a specular term, blended by the metalness parameter $m$ [S1, unofficial]:

$$ f_r(\omega_i, \omega_o) = (1 - m) f_{diff} + m f_{spec} $$

The diffuse term uses a simplified Lambertian model modulated by the diffuse albedo [S1, unofficial]:

$$ f_{diff} = \frac{a(\mathbf{x})}{\pi} $$

The specular term follows the Cook-Torrance model [S1, unofficial]:

$$ f_{spec} = \frac{D(\mathbf{h}) F(\omega_i, \mathbf{h}) G(\omega_i, \omega_o, \mathbf{h})}{4 (\mathbf{n} \cdot \omega_i) (\mathbf{n} \cdot \omega_o)} $$

Where:
*   $\mathbf{h} = \frac{\omega_i + \omega_o}{\|\omega_i + \omega_o\|}$ is the half-vector.
*   $D(\mathbf{h})$ is the Normal Distribution Function (NDF). The sources cite the Disney BRDF but do not provide the explicit NDF formula (e.g., GGX or Beckmann) in the provided text [S1, unofficial]. [G] Standard Disney implementations typically use a GGX distribution: $D(\mathbf{h}) = \frac{\alpha^2}{\pi ((\mathbf{n} \cdot \mathbf{h})^2 (\alpha^2 - 1) + 1)^2}$, where $\alpha$ is derived from roughness $\beta$.
*   $F(\omega_i, \mathbf{h})$ is the Fresnel term. The sources do not specify the exact Fresnel equation (e.g., Schlick approximation) [S1, unofficial]. [G] Typically $F(\omega_i, \mathbf{h}) = F_0 + (1 - F_0)(1 - \omega_i \cdot \mathbf{h})^5$, where $F_0$ is the specular albedo $s$ for dielectrics and interpolated for metals.
*   $G(\omega_i, \omega_o, \mathbf{h})$ is the geometry function. The sources do not specify the exact geometry function (e.g., Schlick-GGX) [S1, unofficial]. [G] Typically $G(\omega_i, \omega_o, \mathbf{h}) = G_{Schlick}(\omega_i, \mathbf{n}, \alpha) G_{Schlick}(\omega_o, \mathbf{n}, \alpha)$.

**Steps:**
1.  Retrieve diffuse albedo $a(\mathbf{x})$ from the spatial texture at point $\mathbf{x}$ [S1, unofficial].
2.  Compute the half-vector $\mathbf{h}$ from $\omega_i$ and $\omega_o$ [S1, unofficial].
3.  Evaluate the NDF $D(\mathbf{h})$ using roughness $\beta$ [S1, unofficial].
4.  Evaluate the Fresnel term $F(\omega_i, \mathbf{h})$ using specular albedo $s$ and metalness $m$ [S1, unofficial].
5.  Evaluate the geometry term $G(\omega_i, \omega_o, \mathbf{h})$ using roughness $\beta$ [S1, unofficial].
6.  Compute the specular BRDF value $f_{spec}$ using the Cook-Torrance formula [S1, unofficial].
7.  Compute the diffuse BRDF value $f_{diff}$ using $a(\mathbf{x})$ [S1, unofficial].
8.  Blend the terms: $f_r = (1 - m) f_{diff} + m f_{spec}$ [S1, unofficial].
9.  Multiply $f_r$ by the incident radiance $L_i(\omega_i)$ and integrate over the hemisphere (typically via Monte Carlo sampling) [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def isotropic_disney_brdf(x, omega_i, omega_o, n, a_texture, s, beta, m):
    # 1. Retrieve diffuse albedo
    a = a_texture.sample(x) # Spatially varying [S1, unofficial]
    
    # 2. Compute half vector
    h = normalize(omega_i + omega_o)
    
    # 3. Evaluate NDF D(h)
    # Note: Exact NDF formula not provided in sources [S1, unofficial]
    # Assuming standard GGX implementation [G]
    alpha = beta # Or mapped from beta
    nh = dot(n, h)
    if nh <= 0:
        return 0.0
    denom = (nh * nh) * (alpha * alpha - 1) + 1
    D = (alpha * alpha) / (pi * denom * denom)
    
    # 4. Evaluate Fresnel F(omega_i, h)
    # Note: Exact Fresnel formula not provided in sources [S1, unofficial]
    # Assuming Schlick approximation [G]
    # F0 depends on metalness m and specular albedo s
    F0 = mix(vec3(s), a, m) # Interpolate between specular albedo and diffuse albedo
    F = F0 + (1 - F0) * pow(1 - dot(omega_i, h), 5)
    
    # 5. Evaluate Geometry G(omega_i, omega_o, h)
    # Note: Exact Geometry formula not provided in sources [S1, unofficial]
    # Assuming Schlick-GGX [G]
    def schlick_ggx(v, n, alpha):
        nv = dot(v, n)
        if nv <= 0: return 0.0
        r = alpha + (1 - alpha) * nv * nv
        return nv / (nv * (1 - r) + r)
    
    G = schlick_ggx(omega_i, n, alpha) * schlick_ggx(omega_o, n, alpha)
    
    # 6. Compute specular term
    no_i = dot(n, omega_i)
    no_o = dot(n, omega_o)
    if no_i <= 0 or no_o <= 0:
        return 0.0
    
    f_spec = (D * F * G) / (4 * no_i * no_o)
    
    # 7. Compute diffuse term
    f_diff = a / pi
    
    # 8. Blend
    f_r = (1 - m) * f_diff + m * f_spec
    
    return f_r
```

**Parameters:**
*   $a(\mathbf{x})$: Diffuse albedo, range $[0, 1]^3$ [S1, unofficial].
*   $s$: Specular albedo, range $[0, 1]$ [S1, unofficial].
*   $\beta$: Roughness, range $[0, 1]$ [S1, unofficial].
*   $m$: Metalness, range $[0, 1]$ [S1, unofficial].

**Edge Cases:**
*   If $\mathbf{n} \cdot \omega_i \le 0$ or $\mathbf{n} \cdot \omega_o \le 0$, the BRDF is zero (backface or grazing) [G].
*   Precision issues may arise when $\beta$ is very small (mirror-like) or very large (rough) [S1, unofficial].

## Subtopics

### Specular ABSDF Factorization
A method for factorizing the specular component of the BRDF, used in conjunction with this model for differentiable rendering [10872-specular-absdf-factorization.md].

### Phys3DGS BRDF Function
A physically-based BRDF function for 3D Gaussian Splatting, offering an alternative shading model [11071-phys3dgs-brdf-function.md].

## Operating characteristics
*   **Complexity:** The evaluation is $O(1)$ per sample, but the cost depends on the complexity of the NDF, Fresnel, and Geometry functions, which are not fully specified in the sources [S1, unofficial].
*   **Performance:** The model is described as "simplified" and "robust to roughness variations" compared to more complex basis-based methods [S1, unofficial]. It is used in Monte Carlo shading, requiring $N$ samples for convergence [S1, unofficial].
*   **Strengths:** Unified framework for metals and plastics; intuitive parameters; reduced ambiguity in optimization by restricting to opaque surfaces [S1, unofficial][S2].
*   **Limitations:** Simplified; does not include all features of the full Disney Principled BRDF (e.g., clearcoat, sheen) [S2]; assumes isotropy [S1, unofficial].
*   **When to use:** When optimizing material parameters for opaque surfaces in differentiable rendering pipelines [S2].

## Background
The Isotropic Disney BRDF is based on the Cook-Torrance model [S1, unofficial]. It is derived from the Disney Principled BRDF, which is widely adopted in movies and games for its versatile appearance and intuitive controls [S2]. The specific simplified version used here is employed in differentiable rendering contexts, such as DIB-R++ and texture-space differentiable rendering, to facilitate learning and optimization of lighting and material parameters [S1, unofficial][S2].

## Sources
[S1] DIB-R++: Learning to Predict Lighting and Material with a Hybrid Differentiable Renderer, https://doi.org/10.48550/arxiv.2111.00140, fetched 2026-10-26 (unofficial)
[S2] Material and Lighting Reconstruction for Complex Indoor Scenes with Texture-space Differentiable Rendering, https://doi.org/10.2312/sr.20211292, fetched 2026-10-29
