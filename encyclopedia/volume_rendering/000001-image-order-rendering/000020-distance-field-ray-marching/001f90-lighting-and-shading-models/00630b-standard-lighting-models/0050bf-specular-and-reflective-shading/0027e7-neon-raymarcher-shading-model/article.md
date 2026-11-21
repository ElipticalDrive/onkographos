# Neon Raymarcher Shading Model

Also called: Milestone 2 shading, Procedural Neon Shading

## Overview
The Neon Raymarcher Shading Model is a real-time shading technique designed for procedural neon city scenes, computing diffuse, specular, ambient occlusion, metallic reflection, and emissive contributions per pixel [S1, unofficial]. It distinguishes itself from siblings like [Ray Marching with Reflections](1963-ray-marching-with-reflections.md) by integrating a specific metallic Fresnel weighting scheme and a fixed-step shadow ray implementation optimized for the compressed distance field of the neon city geometry, rather than relying on general-purpose path tracing or complex environment maps.

## How it works
**Inputs and Outputs:**
*   **Inputs:** Surface point $\mathbf{p}$, surface normal $\mathbf{n}$, view direction $\mathbf{v}$ (from $\mathbf{p}$ to camera, denoted as $\mathbf{rd}$ in source), light direction $\mathbf{l}$ (from $\mathbf{p}$ to light, denoted as $\mathbf{lightDir}$), material properties: albedo ($\mathbf{c}_{albedo}$), metalness ($m \in [0,1]$), emissive color ($\mathbf{c}_{emissive}$), and light color ($\mathbf{c}_{light}$).
*   **Outputs:** Final pixel color $\mathbf{c}_{final}$.

**Data Structures:**
The material state `sf` contains:
*   `albedo`: `vec3`
*   `metal`: `float` (0.0 to 1.0)
*   `emissive`: `vec3`

**Steps:**

1.  **Geometric Setup:**
    Calculate the half-vector $\mathbf{h}$ between the view direction $\mathbf{rd}$ and light direction $\mathbf{lightDir}$:
    $$ \mathbf{h} = \text{normalize}(\mathbf{lightDir} - \mathbf{rd}) $$
    Calculate the dot product for the diffuse term:
    $$ d_{diff} = \max(\mathbf{n} \cdot \mathbf{lightDir}, 0.0) $$
    Calculate the dot product for the specular term:
    $$ d_{spec} = \max(\mathbf{n} \cdot \mathbf{h}, 0.0) $$
    Calculate the dot product for the Fresnel term:
    $$ d_{fresnel\_dot} = \max(\mathbf{n} \cdot (-\mathbf{rd}), 0.0) $$

2.  **Shadow Calculation:**
    Compute the shadow factor $S$ by casting a shadow ray. The ray origin is offset along the normal to prevent self-shadowing artifacts:
    $$ \mathbf{p}_{shadow} = \mathbf{p} + \mathbf{n} \times 0.03 $$
    The shadow function `Shadow(p, lightDir, maxDist)` returns a value in $[0, 1]$ indicating occlusion. The source uses a fixed maximum distance of 12.0 units [S1, unofficial].
    $$ S = \text{Shadow}(\mathbf{p}_{shadow}, \mathbf{lightDir}, 12.0) $$
    The effective diffuse lighting is modulated by this shadow:
    $$ L_{diff} = d_{diff} \times S $$

3.  **Ambient Occlusion Calculation:**
    Compute the ambient occlusion factor $A$ using the function `calcAO(p, n)` [S1, unofficial]. This function estimates the visibility of the hemisphere around $\mathbf{p}$ based on the geometry.

4.  **Specular Reflection:**
    Calculate the specular exponent $E$ and intensity $I_{spec}$ based on the metalness $m$:
    $$ E = \text{mix}(32.0, 140.0, m) $$
    $$ I_{spec} = \text{mix}(0.4, 2.0, m) $$
    The specular contribution is:
    $$ C_{spec} = \text{pow}(d_{spec}, E) \times I_{spec} \times S $$

5.  **Metallic Reflection:**
    Calculate the Fresnel factor $F$ using a power of 5.0:
    $$ F = \text{pow}(1.0 - d_{fresnel\_dot}, 5.0) $$
    Get the sky color $\mathbf{c}_{sky}$ by reflecting the view direction $\mathbf{rd}$ about the normal $\mathbf{n}$:
    $$ \mathbf{r} = \text{reflect}(\mathbf{rd}, \mathbf{n}) $$
    $$ \mathbf{c}_{sky} = \text{skyColor}(\mathbf{r}) $$
    Calculate the metallic reflection intensity $I_{metal}$:
    $$ I_{metal} = \text{mix}(0.03, 0.6, m) \times (0.25 + 0.75 \times F) $$
    The metallic reflection contribution is:
    $$ C_{metal} = \mathbf{c}_{sky} \times I_{metal} \times A $$

6.  **Color Accumulation:**
    Combine the components. The base color is the albedo modulated by ambient light and AO, plus the diffuse light contribution. The specular and metallic reflections are added. Finally, the emissive color is added.
    $$ \mathbf{c}_{final} = \mathbf{c}_{albedo} \times (\text{ambient} \times A) + \mathbf{c}_{albedo} \times \mathbf{c}_{light} \times L_{diff} + \mathbf{c}_{light} \times C_{spec} + C_{metal} + \mathbf{c}_{emissive} $$
    Note: The source code snippet implies `ambient` is a scalar constant or uniform value, and `lightCol` is $\mathbf{c}_{light}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neon_raymarcher_shading(p, n, rd, lightDir, lightCol, sf, ambient):
    """
    p: vec3, surface point
    n: vec3, surface normal
    rd: vec3, view direction (from p to camera)
    lightDir: vec3, light direction (from p to light)
    lightCol: vec3, light color
    sf: struct with albedo (vec3), metal (float), emissive (vec3)
    ambient: float, ambient light intensity
    """
    # 1. Geometric Setup
    halfVec = normalize(lightDir - rd)
    diff_dot = max(dot(n, lightDir), 0.0)
    spec_dot = max(dot(n, halfVec), 0.0)
    fresnel_dot = max(dot(n, -rd), 0.0)

    # 2. Shadow Calculation
    # Offset normal to prevent self-shadowing
    p_shadow = p + n * 0.03
    shadow = shadow_ray(p_shadow, lightDir, 12.0)
    diff_term = diff_dot * shadow

    # 3. Ambient Occlusion
    ao = calc_ao(p, n)

    # 4. Specular Reflection
    # Mix specular exponent and intensity based on metalness
    spec_exp = mix(32.0, 140.0, sf.metal)
    spec_int = mix(0.4, 2.0, sf.metal)
    spec_contrib = pow(spec_dot, spec_exp) * spec_int * shadow

    # 5. Metallic Reflection
    # Fresnel factor
    fresnel = pow(1.0 - fresnel_dot, 5.0)
    
    # Reflected view direction
    r = reflect(rd, n)
    sky_color = sky_color_function(r)
    
    # Metallic reflection intensity
    metal_base = mix(0.03, 0.6, sf.metal)
    metal_grazing_boost = 0.25 + 0.75 * fresnel
    metal_contrib = sky_color * metal_base * metal_grazing_boost * ao

    # 6. Color Accumulation
    # Base color: albedo * (ambient * ao) + albedo * light * diff
    base_color = sf.albedo * (ambient * ao) + sf.albedo * lightCol * diff_term
    
    # Add specular and metallic
    color = base_color + lightCol * spec_contrib + metal_contrib
    
    # Add emissive
    color += sf.emissive
    
    return color
```

**Parameters:**
*   Shadow ray origin offset: 0.03 units along $\mathbf{n}$ [S1, unofficial].
*   Shadow ray max distance: 12.0 units [S1, unofficial].
*   Specular exponent range: 32.0 (non-metal) to 140.0 (metal) [S1, unofficial].
*   Specular intensity range: 0.4 (non-metal) to 2.0 (metal) [S1, unofficial].
*   Fresnel power: 5.0 [S1, unofficial].
*   Metallic reflection base intensity: 0.03 (non-metal) to 0.6 (metal) [S1, unofficial].
*   Metallic reflection grazing boost: 0.25 (normal) to 0.75 (grazing) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-Shadowing:** The normal offset of 0.03 is critical. Without it, the shadow ray may intersect the surface itself, causing dark artifacts. The value is tuned for the specific scale of the neon city geometry [S1, unofficial].
*   **Metalness Interpolation:** The linear interpolation of specular exponent and intensity assumes a linear relationship between material appearance and the `metal` parameter. This is a simplification; physically based models might use different curves.
*   **Sky Color:** The `skyColor` function is not defined in the shading model itself but is an external dependency. It must be implemented to return appropriate environment colors for the reflection.

## Operating characteristics
*   **Time Complexity:** The shading cost is dominated by the `shadow_ray` and `calc_ao` functions, which involve additional distance field evaluations. The number of steps depends on the implementation of these helpers. The core shading formula is $O(1)$.
*   **Memory Complexity:** $O(1)$ per pixel, storing only intermediate vectors and scalars.
*   **Strengths:** Real-time performance on modern hardware due to simplified shading model. The metallic reflection model provides a convincing neon aesthetic with minimal computational overhead compared to full path tracing.
*   **Limitations:** The shadow ray is cast in a fixed direction with a fixed max distance, which may not accurately represent complex lighting scenarios or large scenes. The ambient occlusion approximation may lack detail compared to screen-space or baked solutions. The model is specific to the procedural neon city context and may not generalize well to other materials or scenes.
*   **When to use:** Ideal for real-time rendering of stylized, neon-lit urban environments where performance is critical and a specific aesthetic is desired.
*   **When not to use:** Not suitable for physically accurate rendering, large-scale outdoor scenes with complex lighting, or materials that do not fit the neon/metallic aesthetic.

## Background
The Neon Raymarcher Shading Model originates from the "neon-raymarcher" project by PIGJET, a real-time raymarched demo of a procedural neon city [S1, unofficial]. It was developed as part of a technical demo showcasing the capabilities of distance field ray marching in real-time graphics. The model is designed to complement the specific distance field geometry of the neon city, which includes compressed building distances and a ground plane. The shading model's parameters and structure are tuned to the visual style of the demo, emphasizing bright, reflective surfaces and emissive neon lights.

## Sources
[S1] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
