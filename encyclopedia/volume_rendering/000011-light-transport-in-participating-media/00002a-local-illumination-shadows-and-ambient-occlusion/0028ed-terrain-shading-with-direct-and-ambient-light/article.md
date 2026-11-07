# Terrain Shading with Direct and Ambient Light

Also called: Hemispheric Ambient Occlusion Shading

## Overview
This method computes the final color of a terrain surface point by combining albedo, direct sunlight (modulated by shadow and cloud shadow), and hemispheric ambient light (sky and ground bounce modulated by ambient occlusion). It is designed for heightfield-based terrain rendering where normals are baked rather than derived from finite differences to avoid facet artifacts. It differs from [Direct Illumination with Shadow Rays](2011-direct-illumination-with-shadow-rays.md) by explicitly modeling the hemispheric nature of ambient light (sky vs. ground) and from [Ambient Light Approximation](2244-ambient-light-approximation.md) by using a specific sky/ground interpolation factor based on the surface normal's Y component.

## How it works
**Inputs:**
*   `albedo`: RGB color of the surface material.
*   `sunRadiance`: RGB color of the direct sunlight.
*   `sunDirection`: Normalized 3D vector $(sunX, sunY, sunZ)$.
*   `skyAmbient`: RGB color of the sky ambient light.
*   `groundAmbient`: RGB color of the ground ambient light.
*   `field`: The terrain heightfield data structure, providing access to `cell` size and height sampling.
*   `px, py, pz`: World coordinates of the shading point.
*   `nx, ny, nz`: Normalized surface normal components.

**Outputs:**
*   `out`: RGB final color.

**Data Structures:**
*   The terrain is represented as a grid. The `field.cell` parameter defines the spatial scale used for biasing.
*   Normals are pre-computed (baked) and interpolated, not calculated via finite differences on the fly [S1, unofficial].

**Steps:**

1.  **Direct Illumination Calculation:**
    *   Calculate the dot product of the normal and sun direction:
        $$ NdotL = nx \cdot sunX + ny \cdot sunY + nz \cdot sunZ $$
    *   Calculate the shadow bias to prevent self-shadowing artifacts on coarse heightfields. The bias is defined as:
        $$ bias = field.cell \cdot 0.5 + 0.5 $$
    *   Trace a shadow ray starting from a biased position to determine if the point is in shadow. The start position is:
        $$ \mathbf{p}_{start} = (px + nx \cdot bias, py + bias, pz + nz \cdot bias) $$
        The ray direction is `sunDirection`. A jitter may be applied to the start position or direction to soften shadows [S1, unofficial].
    *   Determine the cloud shadow factor `cloudShade` at the horizontal position $(px, pz)$ and height $py$ [S1, unofficial].
    *   Calculate the direct light scale:
        $$ lightScale = \max(0, NdotL) \cdot shadow \cdot cloudShade $$
        where `shadow` is 1.0 if the shadow ray hits nothing (sunlit) and 0.0 if it hits terrain.

2.  **Ambient Occlusion Calculation:**
    *   Trace an ambient occlusion ray or hemisphere to determine the occlusion factor `occlusion` at $(px, py, pz)$ with normal $(nx, ny, nz)$ [S1, unofficial]. This factor represents the fraction of the hemisphere visible to the sky.

3.  **Hemispheric Ambient Calculation:**
    *   Calculate the sky factor based on the normal's Y component:
        $$ sky = 0.5 + 0.5 \cdot ny $$
    *   Interpolate between sky and ground ambient colors:
        $$ ambientColor = skyAmbient \cdot sky + groundAmbient \cdot (1 - sky) $$
    *   Apply occlusion:
        $$ ambient = ambientColor \cdot occlusion $$

4.  **Final Color Composition:**
    *   Calculate the direct contribution:
        $$ directRGB = sunRadiance \cdot lightScale $$
    *   Calculate the final color:
        $$ out = albedo \cdot (directRGB + ambient) $$

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def shade_terrain(albedo, sun_radiance, sun_direction, sky_ambient, ground_ambient, field, px, py, pz, nx, ny, nz):
    # 1. Direct Illumination
    ndotl = nx * sun_direction.x + ny * sun_direction.y + nz * sun_direction.z
    
    # Bias calculation to prevent self-shadowing
    bias = field.cell * 0.5 + 0.5
    
    # Shadow ray origin with bias
    shadow_origin_x = px + nx * bias
    shadow_origin_y = py + bias
    shadow_origin_z = pz + nz * bias
    
    # Trace shadow ray from shadow_origin in direction sun_direction
    # Returns 1.0 if sun is visible, 0.0 if occluded
    shadow = trace_shadow_ray(shadow_origin_x, shadow_origin_y, shadow_origin_z, sun_direction)
    
    # Cloud shadow factor at horizontal position (px, pz) and height py
    cloud_shade = cloud_shadow_at(px, pz, py)
    
    # Light scale
    light_scale = max(0.0, ndotl) * shadow * cloud_shade
    
    # 2. Ambient Occlusion
    # Trace ambient occlusion for the hemisphere at the point
    occlusion = trace_ambient_occlusion(px, py, pz, nx, ny, nz)
    
    # 3. Hemispheric Ambient
    # Sky factor based on normal Y
    sky_factor = 0.5 + 0.5 * ny
    
    # Interpolate ambient colors
    ambient_r = (sky_ambient.r * sky_factor + ground_ambient.r * (1.0 - sky_factor)) * occlusion
    ambient_g = (sky_ambient.g * sky_factor + ground_ambient.g * (1.0 - sky_factor)) * occlusion
    ambient_b = (sky_ambient.b * sky_factor + ground_ambient.b * (1.0 - sky_factor)) * occlusion
    
    # 4. Final Color
    direct_r = sun_radiance.r * light_scale
    direct_g = sun_radiance.g * light_scale
    direct_b = sun_radiance.b * light_scale
    
    out_r = albedo.r * (direct_r + ambient_r)
    out_g = albedo.g * (direct_g + ambient_g)
    out_b = albedo.b * (direct_b + ambient_b)
    
    return (out_r, out_g, out_b)
```

**Parameters:**
*   `field.cell`: The size of a terrain grid cell. Typical values depend on the world scale.
*   `bias`: Fixed offset of `field.cell * 0.5 + 0.5` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Self-Shadowing:** If the shadow ray origin is not biased sufficiently, coarse heightfields will exhibit severe self-shadowing artifacts (stair-stepping). The bias must be at least half a cell plus a small constant [S1, unofficial].
*   **Normal Accuracy:** Using finite differences to compute normals from a bilinear heightfield results in piecewise constant normals per cell, causing visible facet edges. Normals must be baked or smoothed [S1, unofficial].
*   **Boundary Reads:** Sampling at tile boundaries can lead to out-of-bounds reads if not handled, resulting in NaN values. Guards must be in place for bilinear taps near edges [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of shadow ray samples and ambient occlusion samples. The shadow ray and AO ray tracing are the dominant costs.
*   **Memory Complexity:** Low, requiring only the terrain heightfield and pre-baked normal/attribute maps.
*   **Strengths:** Provides a visually pleasing approximation of global illumination for terrain by separating sky and ground ambient contributions. It is computationally cheaper than full path tracing.
*   **Limitations:** Shadows are hard-edged unless jittered or sampled multiple times. Cloud shadows are decoupled from the terrain geometry. The method assumes a static sun direction for the shading calculation.

## Background
This method is derived from the shading implementation in the `theme1` project by mingderwang, which focuses on procedural terrain generation and rendering [S1, unofficial]. It addresses the specific visual requirements of heightfield-based terrain, where standard surface shading models fail to capture the interplay between sky visibility and ground bounce. The approach was refined to handle the specific artifacts of coarse heightfields, such as self-shadowing and normal discontinuities.

## Sources
[S1] theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
