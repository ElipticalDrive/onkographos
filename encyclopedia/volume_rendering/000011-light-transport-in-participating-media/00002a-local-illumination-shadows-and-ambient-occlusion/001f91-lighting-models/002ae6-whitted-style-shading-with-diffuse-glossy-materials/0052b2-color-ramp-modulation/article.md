# Color Ramp Modulation


## Overview
Modulates diffuse and specular colors using a color ramp texture based on light intensity values [S1, unofficial]. It builds on [Whitted-Style Shading with Diffuse/Glossy Materials](4896-whitted-style-shading-with-diffuse-glossy-materials.md) [G].

## How it works
**Inputs:**
- `m_ColorRamp`: Color ramp texture.
- `light`: 2D vector where `x` is diffuse intensity and `y` is specular intensity.
- `diffuseColor`: Base material diffuse color.
- `specularColor`: Base material specular color.

**Outputs:**
- `modulatedDiffuseColor`: Diffuse color after ramp modulation.
- `modulatedSpecularColor`: Specular color after ramp modulation.

**Data Structures:**
- `ColorRamp`: Texture storing RGB color values.
- `LightValues`: 2D vector `vec2(x, y)`.

**Steps:**
1. Sample color ramp for diffuse: $C_{diff\_mod} = \text{texture2D}(m\_ColorRamp, \text{vec2}(light.x, 0.0)).rgb$ [S1, unofficial].
2. Sample color ramp for specular: $C_{spec\_mod} = \text{texture2D}(m\_ColorRamp, \text{vec2}(light.y, 0.0)).rgb$ [S1, unofficial].
3. Reset light values to 1.0 after modulation: $light.x = 1.0$, $light.y = 1.0$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def color_ramp_modulation(color_ramp_texture, light, diffuse_color, specular_color):
    """
    Modulates diffuse and specular colors using a color ramp texture.
    """
    ramp_diffuse = color_ramp_texture.sample(vec2(light.x, 0.0)).rgb
    ramp_specular = color_ramp_texture.sample(vec2(light.y, 0.0)).rgb
    light.x = 1.0
    light.y = 1.0
    return ramp_diffuse, ramp_specular
```

**Parameters:** Not given in the sources.

**Edge Cases:** Not given in the sources.

## Subtopics

### Lightmap Modulation
Modulates diffuse and specular colors using a lightmap texture [S1, unofficial]. It samples the lightmap using `texCoord2` if `SEPARATE_TEXCOORD` is defined, otherwise `texCoord` [S1, unofficial]. The sampled lightmap color is multiplied with the specular and diffuse colors [S1, unofficial].

### Vertex Lighting
A simplified lighting calculation using pre-computed vertex light values when `VERTEX_LIGHTING` is defined [S1, unofficial]. It extracts light values: `light = vertexLightValues.xy` [S1, unofficial]. If `COLORRAMP` is defined, it modulates diffuse and specular colors using a color ramp texture based on `light.x` and `light.y` [S1, unofficial]. The final RGB is computed as: $\text{FinalRGB} = \text{AmbientSum} \times \text{diffuseColor.rgb} + \text{DiffuseSum.rgb} \times \text{diffuseColor.rgb} \times light.x + \text{SpecularSum} \times \text{specularColor.rgb} \times light.y$ [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] destroflyer/cubes, https://github.com/destroflyer/cubes, fetched 2026-10-26 (unofficial)
