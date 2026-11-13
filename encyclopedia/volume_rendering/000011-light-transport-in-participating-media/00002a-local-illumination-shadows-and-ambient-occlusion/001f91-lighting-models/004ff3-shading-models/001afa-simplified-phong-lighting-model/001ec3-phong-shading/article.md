# Phong Shading

Also called: Phong Illumination, Phong Illumination Model, Phong Model, Phong Reflection Model, Phong Shading Model

## Overview

Phong Shading is a local illumination model that approximates direct lighting by summing ambient, diffuse, and specular components [S2][S3, unofficial][S7, unofficial]. It solves the problem of rendering smooth, shiny surfaces by evaluating the lighting equation per fragment [S3, unofficial][S11, unofficial]. It differs from its siblings by computing specular highlights based on the angle between the view direction and the reflection vector, whereas [Blinn-Phong Shading Model](2239-blinn-phong-shading-model.md) uses the half-vector for computational efficiency, and [Phong Shading Model for Volume Data](1353-phong-shading-model-for-volume-data.md) adapts this to volumetric gradient-based shading [S2][S8, unofficial][S12, unofficial].

## How it works

**Inputs and Outputs:**
*   **Inputs:** Surface position $\vec{p}$, surface normal $\vec{N}$, view direction $\vec{V}$, light direction $\vec{L}$, material properties ($k_a, k_d, k_s, \alpha$), and light properties ($I_a, I_d, I_s$) [S2][S4, unofficial].
*   **Outputs:** Fragment color $C_{final}$ [S2][S13, unofficial].

**Data Structures:**
The material is defined by a structure containing:
*   `k_a`: Ambient reflection coefficient [S2].
*   `k_d`: Diffuse reflection coefficient [S2].
*   `k_s`: Specular reflection coefficient [S2].
*   `alpha` (or `shininess`): Specular exponent [S2][S4, unofficial].
*   `color_a`: Ambient color multiplier [S2].
*   `color_d`: Diffuse color [S2].
*   `color_s`: Specular color [S2].

**Numbered Steps:**

1.  **Vector Normalization:** Ensure $\vec{N}$, $\vec{V}$, and $\vec{L}$ are unit vectors [S2][S6, unofficial].
    $$ \hat{N} = \frac{\vec{N}}{|\vec{N}|}, \quad \hat{V} = \frac{\vec{V}}{|\vec{V}|}, \quad \hat{L} = \frac{\vec{L}}{|\vec{L}|} $$
2.  **Ambient Component:** Calculate the ambient contribution, assuming constant illumination from all directions [S2][S8, unofficial].
    $$ C_{ambient} = k_a \cdot I_a \cdot \text{color}_a $$
3.  **Diffuse Component:** Calculate the Lambertian reflection [S2]. If the surface faces away from the light, the contribution is zero [S2][S8, unofficial].
    $$ \cos \theta_d = \max(\hat{N} \cdot \hat{L}, 0.0) $$
    $$ C_{diffuse} = k_d \cdot I_d \cdot \text{color}_d \cdot \cos \theta_d $$
4.  **Specular Component:** Calculate the specular highlight [S2]. Two common variants exist:
    *   **Original Phong (Reflection Vector):** Compute the reflection vector $\vec{R}$ of the light direction about the normal [S2][S3, unofficial].
        $$ \vec{R} = 2(\hat{N} \cdot \hat{L})\hat{N} - \hat{L} $$
        $$ \cos \theta_s = \max(\hat{R} \cdot \hat{V}, 0.0) $$
        $$ C_{specular} = k_s \cdot I_s \cdot \text{color}_s \cdot (\cos \theta_s)^{\alpha} $$
    *   **Blinn-Phong (Half-Vector):** Compute the halfway vector $\vec{H}$ between $\hat{L}$ and $\hat{V}$ [S2][S8, unofficial].
        $$ \vec{H} = \frac{\hat{L} + \hat{V}}{|\hat{L} + \hat{V}|} $$
        $$ \cos \theta_h = \max(\hat{N} \cdot \vec{H}, 0.0) $$
        $$ C_{specular} = k_s \cdot I_s \cdot \text{color}_s \cdot (\cos \theta_h)^{\alpha} $$
    *Note: The original Phong model uses the reflection vector [S2][S3, unofficial], while Blinn-Phong uses the half-vector [S8, unofficial][S12, unofficial]. This article describes the original Phong method as the primary definition, with Blinn-Phong as a variant [S2][S8, unofficial].*
5.  **Combine:** Sum the components [S2][S13, unofficial].
    $$ C_{final} = C_{ambient} + C_{diffuse} + C_{specular} $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phong_shading(position, normal, view_direction, light_direction, 
                  material, light):
    """
    Computes the Phong shading color for a single light source.
    
    Args:
        position: vec3, world-space position of the fragment.
        normal: vec3, world-space surface normal (assumed normalized).
        view_direction: vec3, world-space direction from fragment to camera (assumed normalized).
        light_direction: vec3, world-space direction from fragment to light source (assumed normalized).
        material: dict with keys 'ka', 'kd', 'ks', 'shininess', 'color_a', 'color_d', 'color_s'.
        light: dict with keys 'ambient_color', 'diffuse_color', 'specular_color'.
        
    Returns:
        vec3, the final fragment color.
    """
    # 1. Ambient Component
    ambient = material['ka'] * light['ambient_color'] * material['color_a']
    
    # 2. Diffuse Component
    # Clamp dot product to [0, 1] to handle back-facing surfaces
    diff_dot = max(dot(normal, light_direction), 0.0)
    diffuse = material['kd'] * light['diffuse_color'] * material['color_d'] * diff_dot
    
    # 3. Specular Component (Original Phong: Reflection Vector)
    # Calculate reflection vector R = 2 * (N . L) * N - L
    n_dot_l = dot(normal, light_direction)
    reflection = 2.0 * n_dot_l * normal - light_direction
    
    # View vector is already normalized
    # Calculate dot product between reflection and view direction
    spec_dot = max(dot(reflection, view_direction), 0.0)
    
    # Raise to shininess power
    spec_intensity = pow(spec_dot, material['shininess'])
    specular = material['ks'] * light['specular_color'] * material['color_s'] * spec_intensity
    
    # 4. Combine
    color = ambient + diffuse + specular
    
    return color
```

**Parameters with Typical Values:**
*   `shininess` ($\alpha$): Controls the size of the specular highlight. Higher values produce smaller, sharper highlights [S2][S4, unofficial]. Typical values are 30 [S8, unofficial], 64.0 [S6, unofficial], and 100.0 [S11, unofficial].
*   `ka`, `kd`, `ks`: Reflection coefficients [S2].
*   `color_a`, `color_d`, `color_s`: Material colors [S2]. The ambient and diffuse object colors are often assumed identical, and the specular color is a convex combination of the diffuse color and white [S2].

**Edge Cases and Pitfalls:**
*   **Non-Physical:** The model is an approximation and does not conserve energy or strictly follow the rendering equation [S4, unofficial].
*   **Glossy Term Discard:** The original derivation discards the $\cos \theta_s$ factor for the specular term because specular reflections are unaffected by projected surface area [S4, unofficial].
*   **Precision:** The specular term involves a power function; very high shininess values can lead to precision issues if the dot product is slightly less than 1.0 [G].
*   **Back-facing:** The diffuse term must be clamped to 0 for back-facing surfaces to avoid incorrect lighting [S2][S8, unofficial].

## Subtopics

### [Reflectance Map Shading](3606-reflectance-map-shading.md)
Reflectance Map Shading is a technique that evaluates the Phong shading model per-pixel using cube map dependent texture reads, thereby avoiding the need to normalize vectors. It serves as an optimization to reduce lookup table size by replacing per-pixel dot-product calculations with a direct lookup into a pre-computed reflectance map organized as six $n \times n$ tables.

### [Phong Shading with Exponential Fog](4998-phong-shading-with-exponential-fog.md)
A fragment shading model that combines ambient, diffuse, and specular lighting components with exponential distance fog to blend the computed surface color with a fog color based on the fragment's distance from the camera. It uses an exponential decay function for fog attenuation instead of linear fog, as linear fog is commented out in the reference implementation.

## Operating characteristics

*   **Time Complexity:** $O(1)$ per fragment per light source [G]. The cost is dominated by a few dot products, a power function, and vector arithmetic [G].
*   **Memory Complexity:** $O(1)$ for material and light parameters [G].
*   **Measured Performance:** On modern GPUs, Phong shading is extremely fast [G].
*   **Scaling:** Scales linearly with the number of lights [G].
*   **Strengths:** Simple to implement, computationally efficient, produces visually pleasing results for shiny surfaces [S2][S3, unofficial].
*   **Limitations:** Not physically based; does not account for shadows, occlusion, or interreflection [S4, unofficial].
*   **When to use:** Real-time rendering where performance is critical and physical accuracy is not required [S3, unofficial].

## Background

Phong shading was developed by Bui Tuong Phong in 1975 [S4, unofficial]. It was introduced as a method to generate higher quality images with lighting [S1, unofficial][S3, unofficial]. It became the de facto baseline for local illumination in computer graphics [S4, unofficial]. The model is based on empirical observations of how light reflects off surfaces, combining ambient, diffuse, and specular components [S2][S7, unofficial].

## Sources

[S1] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S2] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S3] Antara: An Interactive 3D Volume Rendering and Visualization Framework, https://arxiv.org/abs/1812.04233v1, fetched 2026-10-26 (unofficial)
[S4] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S5] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S6] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S7] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S8] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S9] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S10] [PDF] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S11] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
[S12] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S13] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
