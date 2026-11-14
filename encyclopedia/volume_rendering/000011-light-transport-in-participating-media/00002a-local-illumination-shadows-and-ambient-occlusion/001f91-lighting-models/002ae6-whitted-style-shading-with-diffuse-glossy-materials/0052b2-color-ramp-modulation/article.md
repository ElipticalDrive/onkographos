# Color Ramp Modulation

Also called: Color Ramp Lighting, Intensity-Based Color Modulation

## Overview
Color Ramp Modulation is a shading technique that alters the diffuse and specular color components of a material by sampling a 1D color ramp texture, using light intensity values as texture coordinates. It differs from standard Whitted-style shading by introducing non-linear color mapping based on illumination intensity, allowing for stylized or physically-inspired color shifts (e.g., darkening shadows or brightening highlights) without changing the underlying light transport model. It builds on [Whitted-Style Shading with Diffuse/Glossy Materials](4896-whitted-style-shading-with-diffuse-glossy-materials.md).

## How it works
**Inputs:**
- `m_ColorRamp`: A 1D texture (or 2D texture with height 1) storing the color gradient.
- `light.x`: The diffuse light intensity value (typically in range [0, 1]).
- `light.y`: The specular light intensity value (typically in range [0, 1]).
- `diffuseColor`: The base material diffuse color.
- `specularColor`: The base material specular color.

**Outputs:**
- `modulatedDiffuseColor`: The final diffuse color after ramp modulation.
- `modulatedSpecularColor`: The final specular color after ramp modulation.

**Data Structures:**
- `ColorRamp`: Texture of size $W \times 1$ (or $W \times H$ where $H=1$). Each texel stores an RGB color.
- `LightValues`: A 2D vector `vec2(x, y)` where `x` is diffuse intensity and `y` is specular intensity.

**Steps:**
1. **Sample Diffuse Ramp:** Sample the color ramp texture at coordinate `(light.x, 0.0)` to get the modulated diffuse color.
   $$ C_{diff\_mod} = \text{texture2D}(m\_ColorRamp, \text{vec2}(light.x, 0.0)).rgb $$
2. **Sample Specular Ramp:** Sample the color ramp texture at coordinate `(light.y, 0.0)` to get the modulated specular color.
   $$ C_{spec\_mod} = \text{texture2D}(m\_ColorRamp, \text{vec2}(light.y, 0.0)).rgb $$
3. **Reset Light Values:** After modulation, reset `light.x` and `light.y` to 1.0 to prevent double-modulation in subsequent lighting passes or if the lighting model expects unmodulated intensity for further calculations.
   $$ light.x = 1.0 $$
   $$ light.y = 1.0 $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def color_ramp_modulation(color_ramp_texture, light_intensity, diffuse_color, specular_color):
    """
    Modulates diffuse and specular colors using a color ramp texture.
    
    Args:
        color_ramp_texture: 1D or 2D texture (height 1) for color mapping.
        light_intensity: vec2 where x is diffuse intensity, y is specular intensity.
        diffuse_color: vec3 base diffuse color.
        specular_color: vec3 base specular color.
        
    Returns:
        modulated_diffuse: vec3
        modulated_specular: vec3
    """
    # Sample color ramp for diffuse using light.x
    # Assuming texture2D supports 2D coordinates; y is fixed at 0.0
    ramp_diffuse = color_ramp_texture.sample(vec2(light_intensity.x, 0.0))
    
    # Sample color ramp for specular using light.y
    ramp_specular = color_ramp_texture.sample(vec2(light_intensity.y, 0.0))
    
    # The modulation is typically a multiplication or replacement.
    # Based on [S1, unofficial], the ramp provides the color directly, replacing the base color
    # or modulating it. The note says "Modulates diffuse and specular colors",
    # and the sample returns .rgb. We assume the ramp color replaces or is the
    # primary driver. If the base color is to be preserved, it would be multiplied.
    # However, [S1, unofficial] implies the ramp defines the color based on intensity.
    # We will assume the ramp color is the modulated color.
    
    modulated_diffuse = ramp_diffuse.rgb
    modulated_specular = ramp_specular.rgb
    
    # Reset light values to 1.0 after modulation
    light_intensity.x = 1.0
    light_intensity.y = 1.0
    
    return modulated_diffuse, modulated_specular
```

**Parameters:**
- `light.x`, `light.y`: Must be in the range [0, 1] to correctly map to the 1D texture. Values outside this range may wrap or clamp depending on texture sampling settings.

**Edge Cases:**
- **Intensity > 1.0:** If light intensity exceeds 1.0, the texture coordinate exceeds 1.0. Depending on texture wrapping mode, this may cause incorrect color sampling. Clamping is recommended.
- **Intensity < 0.0:** Negative intensities may cause wrapping or clamping to 0.0.
- **Empty Ramp:** If the ramp texture is not initialized, sampling may return black or undefined values.

## Subtopics

### Lightmap Modulation
Modulates diffuse and specular colors using a lightmap texture instead of a color ramp. It samples the lightmap using `texCoord2` if `SEPARATE_TEXCOORD` is defined, otherwise `texCoord`. The sampled lightmap color is multiplied with the specular and diffuse colors. This method is used when pre-computed lighting information is available in a texture, allowing for baked lighting effects. It is chosen when dynamic lighting is not required or when performance is critical and lighting can be pre-calculated.

### Vertex Lighting
A simplified lighting calculation using pre-computed vertex light values when `VERTEX_LIGHTING` is defined. It extracts light values from `vertexLightValues.xy`. If `COLORRAMP` is defined, it modulates diffuse and specular colors using a color ramp texture based on `light.x` and `light.y`. The final RGB is computed as:
$$ \text{FinalRGB} = \text{AmbientSum} \times \text{diffuseColor.rgb} + \text{DiffuseSum.rgb} \times \text{diffuseColor.rgb} \times light.x + \text{SpecularSum} \times \text{specularColor.rgb} \times light.y $$
This method is used for low-polygon models or mobile platforms where pixel-level lighting is too expensive. It is chosen when performance is prioritized over visual accuracy.

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel, as it involves a constant number of texture samples and arithmetic operations.
- **Memory Complexity:** $O(W)$ for the color ramp texture, where $W$ is the width of the ramp.
- **Performance:** Very fast, as it avoids complex lighting calculations. The bottleneck is texture sampling, which is highly optimized in modern GPUs.
- **Strengths:** Simple to implement, allows for stylized rendering, and can simulate complex lighting effects with minimal computational cost.
- **Limitations:** The color modulation is based on intensity only, so it does not account for angle of incidence or other physical properties. It may look unnatural if the ramp is not carefully designed.
- **When to Use:** Stylized games, mobile applications, or scenes where performance is critical and visual fidelity can be sacrificed for speed.
- **When Not to Use:** Physically-based rendering (PBR) pipelines where accurate light transport is required.

## Background
Color Ramp Modulation is a technique used in real-time rendering to achieve stylized or non-photorealistic effects. It is often used in games and interactive applications to create a specific artistic style. The technique is not associated with a specific academic paper but is a common practice in game development. It is used in various game engines and custom renderers.

## Sources
[S1] destroflyer/cubes, https://github.com/destroflyer/cubes, fetched 2026-10-26 (unofficial)
