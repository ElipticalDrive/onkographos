# Spherical Gaussian Shading

Also called: SG Shading, Shading Rendering of Spherical Gaussians, Specular BRDF Approximation

## Overview
Spherical Gaussian (SG) Shading approximates complex lighting environments and specular reflections by decomposing incident radiance into a sum of Spherical Gaussians, which are then rendered as discrete point lights or integrated into a microfacet BRDF model [S1, unofficial][S2]. It solves the problem of efficient relighting in scenes with spatially variant lighting, such as Neural Radiance Fields (NeRFs), by avoiding the high computational cost of full spherical harmonic integration [S1, unofficial][S2]. Unlike [Diffuse Lighting Models](5343-diffuse-lighting-models.md) which ignore view-dependent effects, SG Shading explicitly models specular highlights and view-dependent reflectance [S1, unofficial].

## How it works
The method operates on a surface point $\mathbf{p}$ (or ray origin $\mathbf{r}$) and its geometric data, following the shared principles of [Lighting and Shading Models](2055-lighting-and-shading-models.md).

### Inputs and Outputs
*   **Inputs:** Surface point position $\mathbf{p}$ (or ray $\mathbf{r}$) [G]. Surface normal $\mathbf{n}$ [G]. View direction $\mathbf{v}$ (from $\mathbf{p}$ to camera) [G]. Set of Spherical Gaussians $\{L_k\}$ representing incident radiance, or a set of $k$ point lights with intensities $\mathbf{I}_i$ and positions [S1, unofficial][S2]. Hyperparameter $\gamma$ for luminance control [S2].
*   **Outputs:** Outgoing radiance $L_o(\mathbf{v})$ or final pixel color $I(\mathbf{r})$ [S1, unofficial][S2].

### Data Structures
*   **Spherical Gaussian ($L_k$):** Defined by a center direction, sharpness parameter, and intensity coefficient [G]. In the point-light approximation variant, each SG is decomposed into 5 point lights: one at the SG center and four sampled around it with intensities diminishing according to Gaussian probabilities [S2].
*   **Point Light:** Defined by position, direction $\mathbf{L}_i(\mathbf{r})$ (vector from light to target), intensity $\mathbf{I}_i$, and distance $\mathbf{Dis}_i(\mathbf{r})$ [S2].

### Steps and Formulas

#### 1. Approximate SG as Point Lights (Spatially Variant Lighting)
When modeling spatially variant lighting (e.g., in NeRF relighting), each Spherical Gaussian is rendered as 5 point lights [S2]. If the environment is High Dynamic Range (HDR), SGs are placed far from the scene to assume distant lighting [S2].

The shading contribution $S_{SG}(\mathbf{r})$ from $k$ point lights is calculated as:
$$ S_{SG}(\mathbf{r}) = \sum_{i=1}^k \frac{\mathbf{N}(\mathbf{r}) \cdot \mathbf{L}_i(\mathbf{r})}{|\mathbf{N}(\mathbf{r})| |\mathbf{L}_i(\mathbf{r})|} \cdot \frac{\mathbf{I}_i}{\gamma \cdot \mathbf{Dis}_i(\mathbf{r})^2} $$
where:
*   $\mathbf{N}(\mathbf{r})$ is the surface normal at the point hit by ray $\mathbf{r}$ [G].
*   $\mathbf{L}_i(\mathbf{r})$ is the direction from the light source to the target point [S2].
*   $\mathbf{I}_i$ is the intensity of the $i$-th light [S2].
*   $\mathbf{Dis}_i(\mathbf{r})$ is the distance between the light and the 3D surface point [S2].
*   $\gamma$ is a hyperparameter controlling luminance [S2].

#### 2. Microfacet BRDF Approximation (Angular Approximation)
For specular response approximation, the outgoing radiance $L_o(\mathbf{v})$ is computed by summing contributions from $k$ SGs representing incident radiance:
$$ L_o(\mathbf{v}) = \sum_k \left[ M(\mathbf{l}_k, \mathbf{v})(\mathbf{n} \cdot \mathbf{l}_k)^+ + \int_{\mathbf{l} \in \Omega} D(\mathbf{l}, \mathbf{v})L_k(\mathbf{l})d\mathbf{l} \right] $$
where:
*   $M(\mathbf{L}_k, \mathbf{v})$ is the factor combining Fresnel and mask-shadowing functions [S1, unofficial].
*   $D(\mathbf{l}, \mathbf{v})$ is the normal distribution function, modeled using an anisotropic Spherical Gaussian [S1, unofficial].
*   $L_k(\mathbf{l})$ is the $k$-th Spherical Gaussian representing incident radiance [S1, unofficial].
*   $(\mathbf{n} \cdot \mathbf{l}_k)^+$ denotes the positive part of the dot product (clamped to 0) [S1, unofficial].

#### 3. Final Compositing (NeRF Relighting)
The final rendering for each ray $\mathbf{r}$ combines diffuse, specular, and SG-shaded components, modulated by shadow visibility:
$$ I(\mathbf{r}) = I_d(\mathbf{r}) + I_s(\mathbf{r}) + V(\mathbf{r})\mathcal{R}(\mathbf{r})S_{SG}(\mathbf{r}) $$
where:
*   $I_d(\mathbf{r})$ is the diffuse component [S2].
*   $I_s(\mathbf{r})$ is the specular component [S2].
*   $V(\mathbf{r})$ is the shadow visibility factor (e.g., from variance shadow mapping) [S2].
*   $\mathcal{R}(\mathbf{r})$ is the material reflectance [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def norm(v):
    return (v[0]**2 + v[1]**2 + v[2]**2)**0.5

def dot(a, b):
    return a[0]*b[0] + a[1]*b[1] + a[2]*b[2]

def spherical_gaussian_shading(surface_point, normal, view_dir, lights, gamma):
    """
    Computes shading using the SG point-light approximation.
    Args:
        surface_point: 3D position vector p.
        normal: Surface normal vector n.
        view_dir: View direction vector v.
        lights: List of light objects, each with:
            - intensity (I_i)
            - position (pos_i)
        gamma: Luminance hyperparameter.
    Returns:
        S_SG: The SG shading contribution.
    """
    S_SG = 0.0
    for light in lights:
        light_to_surface = (surface_point[0] - light.position[0],
                            surface_point[1] - light.position[1],
                            surface_point[2] - light.position[2])
        distance = norm(light_to_surface)
        if distance < 1e-6:
            continue
        L_i = (light_to_surface[0]/distance, light_to_surface[1]/distance, light_to_surface[2]/distance)
        N = (normal[0]/norm(normal), normal[1]/norm(normal), normal[2]/norm(normal))
        cos_theta = dot(N, L_i)
        if cos_theta <= 0:
            continue
        attenuation = light.intensity / (gamma * distance * distance)
        S_SG += cos_theta * attenuation
    return S_SG

def final_rendering(diffuse, specular, shadow_visibility, reflectance, sg_shading):
    """
    Combines components for final pixel color.
    """
    return diffuse + specular + shadow_visibility * reflectance * sg_shading
```

### Parameters and Edge Cases
*   **$\gamma$:** Hyperparameter controlling luminance. Typical values are not specified in the sources.
*   **HDR Lighting:** If the lighting environment is HDR, SGs should be placed far from the scene to approximate distant lighting [S2].
*   **Precision:** Using low-order SGs (e.g., 3rd order) is not precise enough for accurate specular reflection and requires a much higher number of coefficients, making it unusable in real-time scenarios without approximation [S1, unofficial].
*   **Shadowing:** Shadows are handled via variance shadow mapping or depth consistency checks between light and camera views, avoiding cumulative transmittance along shadow rays [S2].

## Subtopics

### Final NeRF Relighting Equation
A specific application of SG Shading within Neural Radiance Fields. It defines the final pixel color as the sum of diffuse ($I_d$), specular ($I_s$), and the SG-shaded component ($S_{SG}$) modulated by shadow visibility ($V$) and reflectance ($\mathcal{R}$) [S2]. This allows for intrinsic decomposition and relighting of NeRF scenes by editing SG parameters per pixel or region [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of lights $k$ or SGs. The point-light approximation (5 lights per SG) increases the constant factor but allows standard rasterization or ray-marching shaders to handle complex lighting [S2].
*   **Memory Complexity:** Low, as it stores only light positions, intensities, and SG coefficients, avoiding large environment maps or precomputed radiance transfer (PRT) tables [S1, unofficial][S2].
*   **Strengths:** Efficient relighting; handles spatially variant lighting; integrates well with NeRF frameworks (e.g., JAX, MipNeRF360) [S2].
*   **Limitations:** Low-order SG approximations lack precision for specular highlights; high precision requires many coefficients, hurting performance [S1, unofficial].
*   **When to use:** When efficient relighting of complex scenes (especially NeRFs) is required, and full path tracing is too slow [S2].
*   **When not to use:** When high-fidelity, physically accurate specular reflections are critical without approximation, as the method requires many coefficients to be precise [S1, unofficial].

## Background
Spherical Gaussian Shading was developed to address the limitations of precomputed global illumination methods, which lack spatial precision, and full spherical harmonic integration, which is computationally expensive [S1, unofficial]. Xu et al. proposed the microfacet BRDF approximation using SGs [S1, unofficial]. Later, the method was adapted for Neural Radiance Fields (NeRFs) to enable efficient relighting and intrinsic decomposition by treating SGs as point lights and using variance shadow mapping for shadows [S2]. The implementation often relies on frameworks like JAX and backbones like MipNeRF360 [S2].

## Sources
[S1] Real-Time Ray Traced Voxel Global Illumination, https://doi.org/10.34626/0vdq-7m27, fetched 2026-10-26 (unofficial)
[S2] Relighting Scenes With Object Insertions in Neural Radiance Fields, https://doi.org/10.1109/TCSVT.2025.3535599, fetched 2026-10-29
