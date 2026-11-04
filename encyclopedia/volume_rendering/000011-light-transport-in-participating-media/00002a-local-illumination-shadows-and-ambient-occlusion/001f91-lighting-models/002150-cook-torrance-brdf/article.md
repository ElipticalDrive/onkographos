# Cook-Torrance BRDF

Also called: GGX Cook-Torrance, Microfacet BRDF, PBR Lighting Model

## Overview
The Cook-Torrance BRDF is a physically based rendering (PBR) lighting model for fully opaque materials that determines how light is reflected by a surface given its microgeometry and material type (metal or dielectric) [S1][S2, unofficial]. It distinguishes between metals and dielectrics, conserves energy, and accurately represents real-world reflectance compared to older models like Phong [S1]. It sets itself apart from siblings like [Shading models](837-shading-models.md) by explicitly modeling surface roughness via microfacets rather than empirical exponents, and from [Whitted-Style Shading with Diffuse/Glossy Materials](4896-whitted-style-shading-with-diffuse-glossy-materials.md) by providing a continuous, physically grounded specular lobe rather than discrete reflection rays.

## How it works
The model computes the outgoing radiance $L_o$ as the sum of emitted light $L_e$ and reflected light, where reflected light is the integral of incoming light $L_i$ multiplied by the bidirectional reflectance distribution function (BRDF) $f_r$ and the cosine of the incident angle [S1]:

$$
L_o = L_e + \int_{\Omega} f_r L_i (\omega_i \cdot n) d\omega_i
$$

where $\Omega$ is the unit hemisphere aligned with the surface normal $n$, $\omega_i$ is the incoming light direction, and $\omega_o$ is the outgoing (view) direction [S1]. The BRDF $f_r$ is a weighted sum of a diffuse term and a specular term [S1]:

$$
f_r = k_d f_{Lambert} + k_s f_{CookTorrance}
$$

where $k_d$ and $k_s$ are the weights for diffuse and specular components, respectively, constrained by $k_d + k_s = 1$ to ensure energy conservation [S1]. The diffuse term is the surface albedo $c$ normalized for integration over the hemisphere:

$$
f_{Lambert} = \frac{c}{\pi}
$$

The specular term $f_{CookTorrance}$ is defined as the product of three functions—Normal Distribution Function ($D$), Fresnel Function ($F$), and Geometry Function ($G$)—divided by four times the dot products of the normal with the outgoing and incoming directions [S1]:

$$
f_{CookTorrance} = \frac{D F G}{4 (\omega_o \cdot n) (\omega_i \cdot n)}
$$

### Inputs and Outputs
**Inputs:**
*   Surface normal $n$ (unit vector).
*   Incoming light direction $\omega_i$ (unit vector, pointing towards the surface).
*   Outgoing view direction $\omega_o$ (unit vector, pointing towards the camera).
*   Roughness parameter $\alpha$ (scalar, typically $0 \le \alpha \le 1$).
*   Base reflectivity $F_0$ (vector, typically RGB).
*   Surface albedo $c$ (vector, typically RGB).
*   Light radiance $L_i$ (vector).
*   Emitted light $L_e$ (vector).

**Outputs:**
*   Outgoing radiance $L_o$ (vector).

### Data Structures
No specific complex data structures are required beyond standard vector types (vec3, vec4) and scalar floats. The light attenuation model uses coefficients $A_{Constant}, A_{Linear}, A_{Quadratic}$ and light radius $r$ [S1].

### Steps
1.  **Calculate Halfway Vector:** Compute the normalized halfway vector $h$ between the light direction $l$ (equivalent to $-\omega_i$ in some conventions, but here defined as incoming light vector) and view vector $v$ (equivalent to $\omega_o$) [S1]:
    $$
    h = \frac{l + v}{\|l + v\|}
    $$
2.  **Compute Normal Distribution Function (D):** Use the Trowbridge-Reitz GGX approximation. The roughness parameter $\alpha$ is often remapped from a user-facing roughness value $r_{user}$ as $\alpha = r_{user}^2$ [S2, unofficial]. The formula is:
    $$
    D_{GGX_{TR}}(n, h, \alpha) = \frac{\alpha^2}{\pi ((n \cdot h)^2 (\alpha^2 - 1) + 1)^2}
    $$
    Note: Source [S1] provides the denominator as $\pi (n \cdot h)^2 (\alpha^2 - 1) + 1$ without the square on the outer parenthesis, but standard GGX/Trowbridge-Reitz formulation [S1, S2 context] implies the squared term in the denominator's main bracket or the square of the whole denominator term. Source [S1] explicitly writes: $D_{GGX_{TR}}(n, h, \alpha) = \frac{\alpha^2}{\pi (n \cdot h)^2 (\alpha^2 - 1) + 1}$. However, standard GGX is $\frac{\alpha^2}{\pi ((n \cdot h)^2 (\alpha^2 - 1) + 1)^2}$. Source [S1] formula (22) is written as: $D_{GGX_{TR}}(n, h, \alpha) = \frac{\alpha^2}{\pi (n \cdot h)^2 (\alpha^2 - 1) + 1}$. This appears to be a typo in the source text missing the square on the denominator's main term or the parenthesis. Standard implementation uses the squared denominator. We follow the standard GGX form for correctness, noting the source discrepancy [S1].
3.  **Compute Geometry Function (G):** Use Schlick-GGX with Smith's method. First, compute the roughness parameter $k$ from $\alpha$:
    $$
    k = \frac{(\alpha + 1)^2}{8}
    $$
    Then compute the geometry attenuation for a vector $i$ (where $i$ is either $v$ or $l$):
    $$
    G_{Schlick}(n, i, k) = \frac{n \cdot i}{(n \cdot i)(1 - k) + k}
    $$
    The total geometry function is the product of the view and light terms:
    $$
    G(n, v, l, \alpha) = G_{Schlick}(n, v, k) G_{Schlick}(n, l, k)
    $$
4.  **Compute Fresnel Function (F):** Use the Fresnel-Schlick approximation. $F_0$ is the base reflectivity (0.04 for non-metals, albedo for metals) [S1]:
    $$
    F(h, v, F_0) = F_0 + (1 - F_0)(1 - h \cdot v)^5
    $$
5.  **Combine Terms:** Calculate the specular BRDF component:
    $$
    f_{specular} = \frac{D F G}{4 (\omega_o \cdot n) (\omega_i \cdot n)}
    $$
    Calculate the diffuse BRDF component:
    $$
    f_{diffuse} = \frac{c}{\pi}
    $$
    Determine weights. The diffuse weight $k_d$ is often derived from energy conservation as $1 - F$ (or $1 - F_{average}$), and $k_s = F$ [S1]. Specifically, $k_d = 1 - F$ and $k_s = F$ ensures conservation [S1].
    $$
    f_r = (1 - F) \frac{c}{\pi} + \frac{D F G}{4 (\omega_o \cdot n) (\omega_i \cdot n)}
    $$
6.  **Calculate Light Radiance $L_i$:** If using a point light with polynomial attenuation, calculate $L_i$ using light color $l_{dehomogenized}$, distance $d$, light radius $r$, and attenuation coefficients $A_{Constant}, A_{Linear}, A_{Quadratic}$ [S1]:
    $$
    L_i = l_{dehomogenized} \frac{A_{Constant} + d A_{Linear} + d^2 A_{Quadratic}}{r - d}
    $$
    where $l_{dehomogenized}$ is the light color vector $(l_r, l_g, l_b, l_w)$ dehomogenized to RGB [S1].
7.  **Calculate Outgoing Light:** Integrate over the hemisphere (or sum for discrete lights):
    $$
    L_o = L_e + \sum_{lights} f_r L_i (\omega_i \cdot n)
    $$

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cook_torrance_brdf(n, l, v, roughness, F0, albedo, Le, lights):
    """
    Compute outgoing radiance using Cook-Torrance BRDF.
    
    Args:
        n: Surface normal (vec3)
        l: Light direction vector (vec3, pointing TO surface)
        v: View direction vector (vec3, pointing TO camera)
        roughness: Scalar roughness parameter (0.0 to 1.0)
        F0: Base reflectivity (vec3)
        albedo: Surface color/albedo (vec3)
        Le: Emitted light (vec3)
        lights: List of light objects with pos, color, radius, attenuation_coeffs
        
    Returns:
        Lo: Outgoing radiance (vec3)
    """
    # Normalize vectors
    n = normalize(n)
    l = normalize(l)
    v = normalize(v)
    
    # Halfway vector
    h = normalize(l + v)
    
    # Dot products
    n_dot_l = dot(n, l)
    n_dot_v = dot(n, v)
    n_dot_h = dot(n, h)
    h_dot_v = dot(h, v)
    
    # Cull backfacing surfaces
    if n_dot_l <= 0 or n_dot_v <= 0:
        return Le

    # Roughness remapping: alpha = roughness^2
    alpha = roughness * roughness
    
    # 1. Normal Distribution Function (GGX/Trowbridge-Reitz)
    # Standard GGX formula: alpha^2 / (pi * ((n.h)^2 * (alpha^2 - 1) + 1)^2)
    denom = (n_dot_h * n_dot_h) * (alpha * alpha - 1.0) + 1.0
    D = (alpha * alpha) / (3.141592653589793 * denom * denom)
    
    # 2. Geometry Function (Schlick-GGX)
    k = (roughness + 1.0) * (roughness + 1.0) / 8.0
    G1_v = n_dot_v / (n_dot_v * (1.0 - k) + k)
    G1_l = n_dot_l / (n_dot_l * (1.0 - k) + k)
    G = G1_v * G1_l
    
    # 3. Fresnel Function (Schlick)
    F = F0 + (1.0 - F0) * pow(1.0 - h_dot_v, 5.0)
    
    # 4. Specular BRDF
    # Denominator: 4 * (n.o) * (n.i)
    # Here n.o is n_dot_v, n.i is n_dot_l
    specular_denom = 4.0 * n_dot_v * n_dot_l
    F_specular = (D * F * G) / specular_denom
    
    # 5. Diffuse BRDF
    # Energy conservation: k_d = 1 - F, k_s = F
    # Diffuse term: (1 - F) * albedo / pi
    diffuse_term = (1.0 - F) * albedo / 3.141592653589793
    
    # Total BRDF
    fr = diffuse_term + F_specular
    
    # 6. Light Contribution
    Lo = Le
    for light in lights:
        # Calculate light radiance Li
        # Distance from surface to light
        # Assuming light.pos is the position of the light source
        # l is direction TO light, so light.pos = surface.pos + l * dist
        # We need distance d. If l is normalized direction, we need d.
        # For point lights, l is usually (light_pos - surface_pos) / dist
        # Here we assume l is provided as normalized direction, so we need d from context.
        # If not provided, we assume d is available or light is directional (d=infinity).
        # For this implementation, we assume d is passed or calculated.
        # Let's assume d is available in light object or calculated.
        d = light.distance_to_surface # Placeholder for actual distance calculation
        
        # Attenuation
        # Li = l_dehomogenized * (A_const + d*A_lin + d^2*A_quad) / (r - d)
        # Note: Source [S1] formula (28) has r - d in denominator.
        # This implies r > d. If d >= r, light is 0.
        if d >= light.radius:
            Li = vec3(0.0)
        else:
            # l_dehomogenized is light.color (RGB)
            # A_const, A_lin, A_quad are light.attenuation_coeffs
            att = light.A_constant + d * light.A_linear + (d * d) * light.A_quadratic
            Li = light.color * (att / (light.radius - d))
        
        # Add contribution: fr * Li * (n . l)
        Lo += fr * Li * n_dot_l
        
    return Lo
```

### Parameters
*   $F_0$: Base reflectivity. Default value is 0.04 for non-metals. Equal to albedo for metallic surfaces (metalness=1) [S1].
*   $\alpha$: Roughness parameter. Often remapped from user input roughness $r$ as $\alpha = r^2$ [S2, unofficial].
*   $A_{Constant}, A_{Linear}, A_{Quadratic}$: Polynomial attenuation model coefficients [S1].
*   $r$: Light radius [S1].

### Edge Cases and Pitfalls
*   **Backfacing:** If $n \cdot l < 0$ or $n \cdot v < 0$, the surface is not visible to the light or camera. Return 0 or emitted light only [S1].
*   **Light Radius:** If distance $d \ge r$, the light contribution is zero to avoid division by zero or negative radiance [S1].
*   **Precision:** At grazing angles ($h \cdot v \to 0$), the Fresnel term approaches 1. Numerical stability in $D$ calculation should handle small $n \cdot h$ values.
*   **Metallic vs Dielectric:** $F_0$ must be set correctly. For metals, $F_0$ is the albedo. For dielectrics, $F_0$ is typically 0.04 [S1].

## Subtopics

### Cook-Torrance GGX BRDF
A physically based rendering BRDF combining Cook-Torrance specular with Lambertian diffuse, featuring anisotropic GGX normal distribution function, Smith height-correlated geometry term, and Schlick Fresnel approximation [S3, unofficial]. It calculates the specular component using anisotropic GGX NDF, geometry term using Smith height-correlated model, and Fresnel term using Schlick approximation [S3, unofficial]. The diffuse component uses Lambertian model with energy-conserving division by $\pi$ [S3, unofficial]. It applies Fresnel-weighted energy conservation between the diffuse and specular lobes and uses roughness-aware Fresnel for environment map reflections [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per light per pixel, assuming a fixed number of lights. The cost is dominated by the evaluation of $D, F, G$ functions.
*   **Memory Complexity:** $O(1)$ per pixel for shading variables.
*   **Performance:** More computationally expensive than Phong due to the evaluation of multiple transcendental functions (pow, sqrt, dot products) [S1].
*   **Strengths:** Physically accurate, energy-conserving, handles a wide range of materials (metals, dielectrics, rough, smooth) [S1][S2, unofficial].
*   **Limitations:** Does not account for subsurface scattering or complex inter-reflections within the material (requires additional models) [S2, unofficial]. Computationally heavier than empirical models.
*   **When to use:** When physically accurate shading is required, especially for metallic and rough surfaces [S1].
*   **When not to use:** For stylized rendering where physical accuracy is not desired, or in real-time applications with extremely limited compute budget where Phong might suffice [S1].

## Background
The Cook-Torrance BRDF is based on the microfacet surface model [S1]. It was introduced by Cook and Torrance in 1982 [S1][S2, unofficial]. It is a foundational model in Physically Based Rendering (PBR) [S1]. It is widely used in modern real-time and offline renderers to simulate realistic lighting [S1][S2, unofficial].

## Sources
[S1] Raymarching Distance Fields with CUDA, https://www.mdpi.com/2079-9292/10/22/2730, fetched 2026-10-27
[S2] Voxel Cone Tracing - GitHub, https://raw.githubusercontent.com/righier/gidemo/master/report.pdf, fetched 2026-10-27 (unofficial)
[S3] NiekMSoftware/bad-apple-voxel-raytracer, https://github.com/NiekMSoftware/bad-apple-voxel-raytracer, fetched 2026-10-24 (unofficial)
