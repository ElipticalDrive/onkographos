# Relightable 3DGS

Also called: Handling Challenging Objects (Reflections/Transparency), Relightable 3D GS, Specular 3D GS

## Overview
Relightable 3DGS extends standard 3D Gaussian Splatting to handle complex appearances such as reflections, transparency, and glossy materials, which standard rasterization approximates poorly [S2]. It embeds physically-based material properties into each Gaussian for deferred shading, decomposing outgoing radiance into a BRDF integration that respects material parameters [S1, unofficial]. This approach enables relighting the entire scene under novel illumination conditions while preserving real-time performance during view synthesis [S1, unofficial].

## How it works
**Inputs:**
*   A set of 3D Gaussians $\{G_i\}$, where each Gaussian $i$ is defined by position $\mathbf{p}_i$, covariance matrix $\Sigma_i$ (or rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), opacity $\alpha_i$, and view-dependent color coefficients (typically spherical harmonics).
*   Material properties per Gaussian: diffuse albedo $f_d$, specular BRDF parameters (roughness $r$), normal $\mathbf{n}_i$, and potentially metallic/roughness maps [S1, unofficial][S2].
*   Lighting environment: Defined by incident radiance $L_i(\omega_i)$ for various directions $\omega_i$ [S1, unofficial].

**Outputs:**
*   Pixel color $C(x,y)$ for the rendered image.

**Data Structures:**
*   **Gaussian Attributes:** Each Gaussian stores standard 3DGS attributes plus:
    *   $\mathbf{n}_i$: Surface normal vector (unit length).
    *   $r_i$: Roughness parameter (scalar, typically $[0, 1]$).
    *   $f_{d,i}$: Diffuse albedo color (RGB).
    *   $f_{s,i}$: Specular BRDF parameters (encoded in $r_i$ and potentially Fresnel base reflectivity).
*   **Lighting Data:** A set of light sources or an environment map providing $L_i(\omega_i)$ [S1, unofficial].

**Steps:**
1.  **Rasterization:** Project 3D Gaussians to 2D screen space. Compute 2D covariance $\Sigma'$ and mean $\mathbf{m}$ for each Gaussian [S1, unofficial]. Sort Gaussians by depth for alpha blending [S1, unofficial].
2.  **Per-Pixel Shading:** For each pixel $(x,y)$, iterate through sorted Gaussians $i$ that contribute to the pixel.
3.  **BRDF Evaluation:** For each contributing Gaussian $i$, compute the specular BRDF $f_s(\omega_0, \omega_i)$ using the Cook-Torrance model [S1, unofficial]:
    $$f_s(\omega_0, \omega_i) = \frac{D(h; r_i) \cdot F(\omega_0, h) \cdot G(\omega_i, \omega_0, h; r_i)}{(\mathbf{n}_i \cdot \omega_i) \cdot (\mathbf{n}_i \cdot \omega_0)}$$
    where:
    *   $\omega_0$ is the view direction from the camera to the Gaussian.
    *   $\omega_i$ is the incident lighting direction from the light source to the Gaussian.
    *   $\mathbf{h} = \frac{\omega_i + \omega_0}{\|\omega_i + \omega_0\|}$ is the half vector [S1, unofficial].
    *   $D(h; r_i)$ is the Normal Distribution Function (NDF), e.g., GGX, parameterized by roughness $r_i$ [S1, unofficial].
    *   $F(\omega_0, h)$ is the Fresnel term, typically using Schlick's approximation [S1, unofficial].
    *   $G(\omega_i, \omega_0, h; r_i)$ is the Geometry term (shadowing/masking), e.g., Smith's method [S1, unofficial].
    *   $\mathbf{n}_i$ is the normal of the Gaussian [S1, unofficial].
4.  **Direct Illumination:** Compute the contribution of direct lights. For point sources, trace secondary rays from the Gaussian to the light to evaluate shadow maps [S1, unofficial]. Calculate the incident radiance $L_i(\omega_i)$ at the Gaussian's position [S1, unofficial].
5.  **PBR Color Calculation:** Compute the outgoing radiance $c'(\omega_0)$ for the Gaussian by integrating the BRDF over the incident lighting directions [S1, unofficial]:
    $$c'(\omega_0) = \sum_{j=0}^{N_s} (f_{d,i} + f_s(\omega_0, \omega_{i,j})) L_{i,j}(\omega_{i,j}) (\omega_{i,j} \cdot \mathbf{n}_i) \Delta \omega_{i,j}$$
    where:
    *   $N_s$ is the number of sampled light directions [S1, unofficial].
    *   $L_{i,j}(\omega_{i,j})$ is the incident radiance from direction $j$ [S1, unofficial].
    *   $\Delta \omega_{i,j}$ is the solid angle associated with the sampled incident direction $j$ [S1, unofficial].
    *   Note: The diffuse term $f_{d,i}$ is typically a constant albedo, while $f_s$ is view-dependent [S1, unofficial].
6.  **Alpha Blending:** Blend the computed color $c'(\omega_0)$ with the accumulated background color using the Gaussian's opacity $\alpha_i$ and the standard alpha blending equation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_relightable_3dgs(gaussians, camera, lights):
    # 1. Project Gaussians to 2D
    projected_gaussians = []
    for g in gaussians:
        # Compute 2D mean and covariance from 3D parameters
        m_2d, sigma_2d = project_gaussian_2d(g, camera)
        projected_gaussians.append({
            'mean': m_2d,
            'cov': sigma_2d,
            'alpha': g.alpha,
            'normal': g.normal,
            'roughness': g.roughness,
            'albedo': g.albedo,
            'depth': g.depth
        })
    
    # 2. Sort by depth
    projected_gaussians.sort(key=lambda g: g['depth'])
    
    # 3. Rasterize and Shade
    image = np.zeros((height, width, 3))
    for y in range(height):
        for x in range(width):
            accumulated_color = np.array([0.0, 0.0, 0.0])
            accumulated_alpha = 0.0
            
            for g in projected_gaussians:
                # Check if pixel is within Gaussian support
                if not is_inside_support(g['mean'], g['cov'], x, y):
                    continue
                
                # Compute 2D Gaussian weight
                weight = compute_gaussian_weight(g['mean'], g['cov'], x, y)
                
                # View direction
                view_dir = compute_view_direction(camera, g['mean'])
                
                # Compute PBR color
                pbr_color = np.array([0.0, 0.0, 0.0])
                for light in lights:
                    light_dir = compute_light_direction(light, g['mean'])
                    half_vec = normalize(light_dir + view_dir)
                    
                    # BRDF components
                    D = compute_ndf(half_vec, g['roughness'])
                    F = compute_fresnel(view_dir, half_vec)
                    G = compute_geometry(light_dir, view_dir, half_vec, g['roughness'])
                    
                    # Specular BRDF
                    denom = max(dot(g['normal'], light_dir) * dot(g['normal'], view_dir), 1e-6)
                    fs = (D * F * G) / denom
                    
                    # Direct illumination with shadowing
                    shadow = evaluate_shadow(g['mean'], light)
                    Li = light.intensity * shadow
                    
                    # PBR contribution
                    pbr_color += (g['albedo'] / pi + fs) * Li * max(dot(g['normal'], light_dir), 0)
                
                # Alpha blending
                a = g['alpha'] * weight
                accumulated_color = accumulated_color * (1 - a) + pbr_color * a
                accumulated_alpha += a * (1 - accumulated_alpha)
                
                if accumulated_alpha > 0.99:
                    break
            
            image[y, x] = accumulated_color
            
    return image
```

**Parameters:**
*   $N_s$: Number of light samples. Not given in the sources.
*   Roughness $r$: Range $[0, 1]$. Typical values depend on material [S1, unofficial].

**Edge Cases:**
*   **Shadow Ray Tracing:** Tracing secondary rays from each Gaussian to the light can be computationally expensive. Optimization strategies like shadow maps are used [S1, unofficial].
*   **Degenerate Normals:** If the normal $\mathbf{n}_i$ is not well-defined (e.g., at boundaries), the BRDF evaluation may produce artifacts [S2].
*   **Precision:** The denominator in the BRDF formula can approach zero if the view or light direction is tangent to the surface, requiring clamping [S1, unofficial].

## Subtopics

### [Deferred Shading with Alpha-Blended Feature Maps](12306-deferred-shading-with-alpha-blended-feature-maps.md)
Deferred Shading with Alpha-Blended Feature Maps is a rendering approach where material properties (albedo, metallic, roughness, normal) are aggregated into pixel-level feature maps via alpha-blending of 2D Gaussians, and the rendering equation is applied in a deferred shading pass. This method treats alpha-blending as a smoothing filter to stabilize optimization and produce cohesive results, unlike shading directly on Gaussians before blending. It differs from per-Gaussian shading by performing complex shading calculations on a per-pixel basis rather than a per-Gaussian basis, which significantly enhances rendering quality and performance for complex materials.

## Operating characteristics
*   **Time Complexity:** Rendering time is dominated by the rasterization of Gaussians and the per-pixel BRDF evaluation. The addition of secondary ray tracing for shadows increases complexity compared to standard 3DGS [S1, unofficial].
*   **Memory Complexity:** Requires storing additional material properties (normal, roughness, albedo) per Gaussian [S1, unofficial].
*   **Strengths:** Handles complex appearances like reflections, transparency, and glossy materials better than standard 3DGS [S2]. Enables relighting under novel illumination [S1, unofficial].
*   **Limitations:** More complex to implement and train than standard 3DGS. Performance may be lower due to BRDF calculations and shadow ray tracing [S1, unofficial]. Robust reconstruction of reflective and transparent objects in general scenes remains challenging [S2].
*   **When to use:** When realistic rendering of complex materials and dynamic lighting is required.
*   **When not to use:** When real-time performance on simple scenes is the primary concern, or when hardware does not support the required shading operations efficiently.

## Background
Relightable 3DGS was introduced to address the limitations of standard 3D Gaussian Splatting in handling complex appearances [S1, unofficial]. Standard 3DGS uses a view-dependent spherical harmonics color model, which fails to represent specular reflections, glossy materials, and global illumination [S1, unofficial]. Relightable 3DGS represents a major paradigm shift by embedding physically-based material properties into each Gaussian for deferred shading [S1, unofficial]. It decomposes the outgoing radiance into a BRDF integration that respects material parameters [S1, unofficial]. This approach allows for relighting the entire scene under novel illumination conditions while preserving real-time performance during view synthesis [S1, unofficial]. Related works include GaussianShader, which simplifies the BRDF to a neural shader, and GS-IR, which performs inverse rendering [S1, unofficial].

## Sources
[S1] 3D Gaussian Splatting Rasterization: A Survey, https://doi.org/10.20944/preprints202607.0776.v2, fetched 2026-10-26 (unofficial)
[S2] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
