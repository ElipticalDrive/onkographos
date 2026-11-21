# Phong Shading

Also called: Phong Illumination, Phong Illumination Model, Phong Model, Phong Reflection Model, Phong Shading Model

## Overview
Phong Shading is a local illumination model that approximates the color of a surface point by summing ambient, diffuse, and specular components based on surface normals, light directions, and view directions [S2][S4, unofficial]. It is distinct from [Phong Shading Model for Volume Data](1353-phong-shading-model-for-volume-data.md) which adapts this model specifically for volume gradients [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Surface position $\vec{x}$, surface normal $\vec{N}$ (unit vector), light direction $\vec{L}$ (unit vector from point to light), view direction $\vec{V}$ (unit vector from point to camera), material parameters ($k_a, k_d, k_s, n$ or $\alpha$), light parameters ($I_a, I_d, I_s$ or colors), and object colors ($O_a, O_d, O_s$) [S6, unofficial].
*   **Outputs:** Final color $I_{total}$ at the surface point [S2].

**Data Structures:**
The material is typically represented as a structure containing:
*   `ka`: Ambient reflection coefficient (scalar) [S2].
*   `kd`: Diffuse reflection coefficient (scalar or vector) [S2].
*   `ks`: Specular reflection coefficient (scalar or vector) [S2].
*   `n` or `alpha`: Shininess exponent (scalar) [S6, unofficial].
*   `color_ambient`: Ambient color vector [S2].
*   `color_diffuse`: Diffuse color vector [S2].
*   `color_specular`: Specular color vector [S2].

**Numbered Steps:**
1.  **Compute Ambient Component:** Calculate the ambient contribution using the ambient light intensity and the material's ambient reflection coefficient [S14, unofficial].
    $$I_{amb} = I_a \cdot k_a \cdot O_a$$
    Where $I_a$ is the ambient light intensity/color, $k_a$ is the ambient reflection coefficient, and $O_a$ is the object's ambient color [S2][S9, unofficial]. Note: Some implementations assume $O_a = O_d$ [S2].

2.  **Compute Diffuse Component:** Calculate the diffuse contribution using Lambert's cosine law [S4, unofficial].
    $$I_{diff} = \sum_{i} I_{i,d} \cdot k_d \cdot O_d \cdot \max(\vec{L}_i \cdot \vec{N}, 0)$$
    Where $I_{i,d}$ is the diffuse intensity of light source $i$, $k_d$ is the diffuse reflection coefficient, $O_d$ is the object's diffuse color, $\vec{L}_i$ is the unit vector from the surface point to light source $i$, and $\vec{N}$ is the unit normal vector [S2][S4, unofficial][S9, unofficial]. If $\vec{L}_i \cdot \vec{N} < 0$, the diffuse term is 0 (back-face culling for lighting) [S9, unofficial][S15, unofficial].

3.  **Compute Specular Component:** Calculate the specular highlight [S14, unofficial]. There are two common variations cited in the sources:
    *   **Reflection Vector Variant (Original Phong):** Compute the reflection vector $\vec{R}$ of the light direction about the normal [S2].
        $$\vec{R} = 2(\vec{N} \cdot \vec{L})\vec{N} - \vec{L}$$
        Then calculate the intensity based on the angle between $\vec{R}$ and the view direction $\vec{V}$:
        $$I_{spec} = \sum_{i} I_i \cdot k_s \cdot O_s \cdot \max(\vec{R} \cdot \vec{V}, 0)^n$$
        Where $I_i$ is the intensity of light source $i$, $k_s$ is the specular reflection coefficient, $O_s$ is the object's specular color, $\vec{V}$ is the unit view vector, and $n$ is the shininess exponent [S2][S4, unofficial][S9, unofficial].
    *   **Half-Vector Variant (Blinn-Phong/Alternative):** Compute the half-vector $\vec{H}$ between the light direction $\vec{L}$ and view direction $\vec{V}$ [S8].
        $$\vec{H} = \frac{\vec{L} + \vec{V}}{\|\vec{L} + \vec{V}\|}$$
        Then calculate the intensity based on the angle between $\vec{H}$ and the normal $\vec{N}$:
        $$I_{spec} = \sum_{i} I_i \cdot k_s \cdot O_s \cdot \max(\vec{H} \cdot \vec{N}, 0)^n$$
        This variant is often attributed to Blinn but is cited in sources discussing the Phong model's implementation details [S2][S8][S10, unofficial][S15, unofficial]. The sources note that the angle between $\vec{R}$ and $\vec{V}$ is twice the angle between $\vec{N}$ and $\vec{H}$ [S2].

4.  **Combine Components:** Sum the ambient, diffuse, and specular components to get the final color [S15, unofficial].
    $$I_{total} = I_{amb} + I_{diff} + I_{spec}$$
    [S2][S4, unofficial][S9, unofficial][S15, unofficial].

**Parameters with Typical Values:**
*   **Shininess ($n$ or $\alpha$):** Controls the size of the specular highlight. Typical values range from 10 to 128. A value of 10 is dull, 20 is shiny, 25 is very shiny/metallic [S2][S6, unofficial][S14, unofficial]. Default often cited as 64.0 or 100.0 [S6, unofficial][S14, unofficial].
*   **Ambient Coefficient ($k_a$):** Typical default values range from 0.05 to 0.3 depending on the material [S2][S14, unofficial].
*   **Diffuse Coefficient ($k_d$):** Typical default 0.7 to 1.0 [S2][S14, unofficial].
*   **Specular Coefficient ($k_s$):** Typical default 0.1 to 1.5 [S2][S14, unofficial].
*   **Light Colors:** Often white (1.0, 1.0, 1.0) for simplicity [S2][S6, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Physical:** The model is not physically sound; it does not conserve energy and $L_{phong}$ does not reflect genuine radiance but an approximated color [S4, unofficial].
*   **Glossy Term Discard:** The specular term discards the $\cos \theta_s$ factor (projected surface area) because specular reflections are considered unaffected by it [S4, unofficial].
*   **Vector Normalization:** All vectors ($\vec{L}, \vec{V}, \vec{N}, \vec{R}, \vec{H}$) must be normalized before dot products to ensure correct intensity [S4, unofficial][S6, unofficial].
*   **Back-Facing Surfaces:** Diffuse and specular terms should be zero if the surface faces away from the light ($\vec{N} \cdot \vec{L} < 0$) [S9, unofficial][S15, unofficial].
*   **Precision:** In volume rendering, normals are computed via gradients (central differences), which can be noisy if the gradient step size is not small enough [S1, unofficial][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def phong_shading(position, normal, light_position, camera_position, material, lights):
    """
    Computes Phong shading for a single surface point.
    
    Args:
        position: vec3, world space position of the point.
        normal: vec3, normalized world space normal.
        light_position: vec3, world space position of the light.
        camera_position: vec3, world space position of the camera.
        material: dict or object with keys: ka, kd, ks, shininess, color_ambient, color_diffuse, color_specular.
        lights: list of dicts, each with keys: color (vec3), position (vec3).
        
    Returns:
        vec3, the final shaded color.
    """
    # Initialize result
    color = vec3(0.0, 0.0, 0.0)
    
    # 1. Ambient Component
    # I_amb = I_a * k_a * O_a
    ambient_light_color = vec3(1.0, 1.0, 1.0) # Default white ambient if not specified in lights
    ambient_contrib = ambient_light_color * material.ka * material.color_ambient
    color += ambient_contrib
    
    for light in lights:
        # 2. Diffuse Component
        # L = normalize(light_position - position)
        light_dir = normalize(light.position - position)
        
        # Diffuse = I_d * k_d * O_d * max(N . L, 0)
        diffuse_intensity = max(dot(normal, light_dir), 0.0)
        diffuse_contrib = light.color * material.kd * material.color_diffuse * diffuse_intensity
        color += diffuse_contrib
        
        # 3. Specular Component
        # V = normalize(camera_position - position)
        view_dir = normalize(camera_position - position)
        
        # Using Half-Vector variant as it is widely cited in the sources for Phong implementation
        half_dir = normalize(light_dir + view_dir)
        specular_intensity = pow(max(dot(normal, half_dir), 0.0), material.shininess)
        
        specular_contrib = light.color * material.ks * material.color_specular * specular_intensity
        color += specular_contrib
        
    return color
```

## Subtopics

### [Reflectance Map Shading](3606-reflectance-map-shading.md)
Reflectance Map Shading is a technique that evaluates the Phong shading model per-pixel using cube map dependent texture reads, thereby avoiding the need to normalize vectors. It serves as an optimization to reduce lookup table size by replacing per-pixel dot-product calculations with a direct lookup into a pre-computed reflectance map organized as six $n \times n$ tables.

### [Phong Shading with Exponential Fog](4998-phong-shading-with-exponential-fog.md)
Phong Shading with Exponential Fog is a local illumination model that combines the standard Phong lighting components (ambient, diffuse, specular) with exponential depth cueing to simulate distance-based fading in volume rendering. It differs from its parent [Phong Shading](1853-phong-shading.md) by explicitly integrating a distance-dependent attenuation factor into the final color calculation, and from [Reflectance Map Shading](3606-reflectance-map-shading.md) by computing lighting dynamically per fragment rather than via precomputed textures.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per surface point per light source [G]. The cost is dominated by vector operations (dot products, normalization, power function) [G].
*   **Memory Complexity:** $O(1)$ for material and light parameters [G].
*   **Measured Performance:** Fast enough for real-time rendering on standard hardware [S4, unofficial].
*   **Scaling:** Linear with the number of light sources.
*   **Strengths:** Computationally inexpensive; produces visually pleasing specular highlights; widely supported and understood [S2][S4, unofficial][S8].
*   **Limitations:** Not physically based; does not account for occlusion or interreflection (local illumination only) [S4, unofficial]; specular highlights can appear sharp or unnatural if the shininess exponent is not tuned correctly [S2].
*   **When to Use:** Real-time rendering, volume visualization where performance is critical, and when approximate lighting is sufficient [S1, unofficial][S4, unofficial][S8].
*   **When Not to Use:** Physically based rendering requiring energy conservation, global illumination effects, or accurate shadowing [S4, unofficial].

## Background
Phong Shading was developed by Bui Tuong Phong in 1975 [S4, unofficial]. It is a standard lighting model applied to SDF-rendered scenes and volume data, using surface positions, normals, material properties, and camera location [S5, unofficial]. It is commonly used in ray casting to approximate lighting [S10, unofficial] and in real-time OpenGL rendering [S4, unofficial]. The model is an empirical approximation used because it can be efficiently computed [S9, unofficial].

## Sources
[S1] GPU-based Compressive Volume Rendering, https://doi.org/10.11575/prism/5366, fetched 2026-10-26 (unofficial)
[S2] Computing and Displaying Isosurfaces inR, https://doi.org/10.18637/jss.v028.i01, fetched 2026-10-27
[S3] Antara: An Interactive 3D Volume Rendering and Visualization Framework, https://arxiv.org/abs/1812.04233v1, fetched 2026-10-26 (unofficial)
[S4] Real-Time Global Illumination Using OpenGL And Voxel Cone Tracing, https://doi.org/10.48550/arxiv.2104.00618, fetched 2026-10-26 (unofficial)
[S5] sungiant/sdf, https://github.com/sungiant/sdf, fetched 2026-10-26 (unofficial)
[S6] skmhrk1209/Torch-Sphere-Tracer, https://github.com/skmhrk1209/Torch-Sphere-Tracer, fetched 2026-10-26 (unofficial)
[S7] skmhrk1209/Sphere-Tracing-PyTorch3D, https://github.com/skmhrk1209/Sphere-Tracing-PyTorch3D, fetched 2026-10-26 (unofficial)
[S8] Enhanced volumetric visualization for real time 4D intraoperative ophthalmic swept-source OCT, https://doi.org/10.1364/boe.7.001815, fetched 2026-11-08
[S9] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
[S10] Massive Data Management for Distributed Volume Visualization, http://hdl.handle.net/1951/47671, fetched 2026-10-26 (unofficial)
[S11] erbenpeter/sdf, https://github.com/erbenpeter/sdf, fetched 2026-10-26 (unofficial)
[S12] [PDF] The VolumePro Real-Time Ray-Casting System, https://www.merl.com/publications/docs/TR99-19.pdf, fetched 2026-10-26 (unofficial)
[S13] PDF Direct Volume Rendering - KTH, https://www.kth.se/social/files/565e35dff27654457fb84363/08_VolumeRendering.pdf, fetched 2026-10-27 (unofficial)
[S14] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
[S15] GPU Ray Marching for Ambient Occlusion, https://www.scribd.com/document/520755804/imm6392, fetched 2026-10-28 (unofficial)
