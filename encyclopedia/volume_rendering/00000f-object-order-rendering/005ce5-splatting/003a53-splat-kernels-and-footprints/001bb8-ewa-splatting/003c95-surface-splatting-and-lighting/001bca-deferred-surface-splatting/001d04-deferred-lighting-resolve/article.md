# Deferred Lighting Resolve

Also called: Final Pass, Lighting Pass

## Overview
Deferred Lighting Resolve applies a lighting model to surface data accumulated during a deferred surface splatting pipeline [S2, unofficial]. It reconstructs the final pixel color by combining weighted color and surface normals from G-buffers with ambient, diffuse, and specular terms [S1, unofficial]. It decouples geometric projection from lighting, allowing screen-space shading without re-projecting geometry [G].

## How it works
**Inputs:**
- G-Buffer textures:
  - `g_colors`: 2D texture storing weighted color sums and accumulation weight [S1, unofficial].
  - `g_normals`: 2D texture storing surface normals [S1, unofficial].
- Uniforms/Constants:
  - `light_dir_view_spc`: View-space light direction vector [S1, unofficial].
  - `light_color`: Light color vector (RGB) [S1, unofficial].
  - `ambient_strength`: Scalar for ambient intensity [S1, unofficial].
  - `specular_strength`: Scalar for specular intensity [S1, unofficial].

**Outputs:**
- Final color buffer (RGBA) [S1, unofficial].

**Data Structures:**
- The G-buffers are 2D textures of size $W \times H$ (screen resolution) [S1, unofficial].
- Internal texture formats and bit packing are not specified in the sources.

**Numbered Steps:**
1. Sample weighted color from the color G-buffer at the current fragment's texture coordinates [S1, unofficial].
2. Discard the fragment if the accumulated weight is zero or negative to avoid division by zero or NaN results [S1, unofficial].
3. Normalize the color by dividing the weighted color RGB by the weight [S1, unofficial].
4. Sample the normal from the normal G-buffer and normalize it [S1, unofficial].
5. Normalize the light direction vector [S1, unofficial].
6. Calculate ambient lighting [S1, unofficial].
7. Calculate diffuse lighting [S1, unofficial].
8. Calculate specular lighting [S1, unofficial].
9. Combine lighting terms with the base color [S1, unofficial].
10. Output the final color to the framebuffer [S1, unofficial].

**Formulas:**
- $C_{\text{weighted}} = \text{texture}(g\_colors, uv)$: Weighted color and accumulated weight [S1, unofficial].
- $C_{\text{base}} = C_{\text{weighted}}.rgb / C_{\text{weighted}}.a$: Final base color [S1, unofficial].
- $N = \text{normalize}(\text{texture}(g\_normals, uv).xyz)$: Surface normal [S1, unofficial].
- $L = \text{normalize}(light\_dir\_view\_spc)$: Normalized light direction [S1, unofficial].
- $C_{\text{ambient}} = \text{ambient\_strength} \times \text{light\_color}$: Ambient term [S1, unofficial].
- $C_{\text{diffuse}} = \max(-\text{dot}(N, L), 0.0) \times \text{light\_color}$: Diffuse term [S1, unofficial].
- $R = \text{reflect}(L, N)$: Reflection vector [S1, unofficial].
- $S_{\text{effect}} = \max(R.z, 0.0)^{32}$: Specular intensity [S1, unofficial].
- $C_{\text{specular}} = \text{specular\_strength} \times S_{\text{effect}} \times \text{light\_color}$: Specular term [S1, unofficial].
- $C_{\text{final}} = (C_{\text{ambient}} + C_{\text{diffuse}} + C_{\text{specular}}) \times C_{\text{base}}$: Combined result [S1, unofficial].
- $\text{FragColor} = \text{vec4}(C_{\text{final}}, 1.0)$: Output color [S1, unofficial].

**Parameters:**
- `ambient_strength`: Default 0.5 [S1, unofficial].
- `specular_strength`: Default 0.2 [S1, unofficial].
- `light_color`: Default (1.0, 1.0, 1.0) [S1, unofficial].
- `shininess`: Fixed at 32 in the reference implementation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If a pixel is not covered by any splat, the weight in `g_colors` is 0. Dividing by this weight results in NaN. The discard step (Step 2) prevents this [S1, unofficial].
- **Light Direction Convention:** The source code uses `max(-dot(normal, light_dir), 0.0)` for diffuse lighting [S1, unofficial]. The exact geometric interpretation of `light_dir_view_spc` is not explicitly defined beyond being a view-space direction.
- **View Space Assumption:** The specular calculation uses the Z-component of the reflection vector, assuming a standard view space where the camera looks down the negative Z-axis [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def deferred_lighting_resolve(uv, g_colors, g_normals, light_dir_view_spc, light_color, ambient_strength, specular_strength):
    # Step 1: Sample weighted color
    color_weighted = g_colors.sample(uv)
    
    # Step 2: Discard if no splat covered this pixel
    if color_weighted.a <= 0.0:
        return None  # discard fragment
    
    # Step 3: Normalize color by weight
    color = color_weighted.rgb / color_weighted.a
    
    # Step 4: Sample and normalize normal
    normal = normalize(g_normals.sample(uv).xyz)
    
    # Step 5: Normalize light direction
    light_dir = normalize(light_dir_view_spc)
    
    # Step 6: Ambient
    ambient = ambient_strength * light_color
    
    # Step 7: Diffuse
    diff_val = max(-dot(normal, light_dir), 0.0)
    diffuse = diff_val * light_color
    
    # Step 8: Specular
    reflection = reflect(light_dir, normal)
    specular_effect = pow(max(reflection.z, 0.0), 32.0)
    specular = specular_strength * specular_effect * light_color
    
    # Step 9: Combine
    result = (ambient + diffuse + specular) * color
    
    # Step 10: Output
    return (result[0], result[1], result[2], 1.0)
```

## Subtopics

### EWA Filter Approximation
An approximation to the original Elliptical Weighted Average (EWA) filter used in the deferred shading pipeline to apply a bandlimiting prefilter [S2, unofficial]. It is controlled by a boolean flag `m_ewa_filter` to enable or disable the approximation [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per pixel for the resolve pass, as it involves a fixed number of texture samples and arithmetic operations [G].
- **Memory Complexity:** $O(W \times H)$ for the G-buffers (depth, color, normal) [G].
- **Performance:** The resolve pass is lightweight compared to the splatting pass. The reference implementation was tested on an NVIDIA GTX 1080 Ti [S2, unofficial].
- **Strengths:** Efficient handling of multiple light sources; decoupling of geometry and lighting [G].
- **Limitations:** Requires G-buffer storage; potential for aliasing if splatting resolution is low; dependent on the quality of the normal accumulation in the splatting pass [G].

## Background
Deferred Lighting Resolve is part of the Surface Splatting technique, specifically the deferred shading pipeline component [S2, unofficial]. It builds on the GPU accelerated surface splatting approach by Botsch et al. [S2, unofficial]. The method is implemented in the open-source project `sebastianlipponer/surface_splatting` [S2, unofficial].

## Sources
[S1] arminkz/PointCloudRendering, https://github.com/arminkz/PointCloudRendering, fetched 2026-10-28 (unofficial)
[S2] sebastianlipponer/surface_splatting, https://github.com/sebastianlipponer/surface_splatting, fetched 2026-10-28 (unofficial)
