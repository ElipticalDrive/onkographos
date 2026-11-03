# Deferred Lighting Resolve

Also called: Final Pass, Lighting Pass

## Overview
Deferred Lighting Resolve is the final shading pass in a deferred surface splatting pipeline [S2, unofficial]. It reconstructs the final pixel color by applying a lighting model (ambient, diffuse, specular) to the weighted color and surface normals accumulated in the G-buffers during the splatting phase [S1, unofficial]. It differs from forward rendering by decoupling the geometric projection (splatting) from the lighting calculation, allowing multiple light sources to be applied efficiently in screen space without re-projecting geometry [G].

## How it works
**Inputs:**
- G-Buffer textures:
  - `g_colors`: RGBA texture storing the weighted sum of splat colors. Layout: `vec4` per pixel [S1, unofficial].
  - `g_normals`: RGBA texture storing the accumulated surface normals. Layout: `vec4` per pixel (XYZ is normal, W is unused or weight) [S1, unofficial].
- Uniforms/Constants:
  - `light_dir_view_spc`: View-space light direction vector [S1, unofficial].
  - `light_color`: Light color vector (RGB) [S1, unofficial].
  - `ambient_strength`: Scalar for ambient intensity [S1, unofficial].
  - `specular_strength`: Scalar for specular intensity [S1, unofficial].

**Outputs:**
- Final color buffer (RGBA) [S1, unofficial].

**Data Structures:**
- The G-buffers are 2D textures of size $W \times H$ (screen resolution) [S1, unofficial].
- `g_colors` uses `GL_RGBA32F` (or equivalent) to store floating-point weighted color sums [S1, unofficial].
- `g_normals` uses `GL_RGBA32F` to store normalized or unnormalized normal vectors [S1, unofficial].

**Steps:**
1. **Sample Weighted Color:** Read the color and weight from the color G-buffer at the current fragment's texture coordinates [S1, unofficial].
   $$ C_{\text{weighted}} = \text{texture}(g\_colors, uv) $$
2. **Discard Empty Pixels:** If the accumulated weight ($C_{\text{weighted}}.a$) is zero or negative, discard the fragment to avoid division by zero or NaN results [S1, unofficial].
   $$ \text{if } C_{\text{weighted}}.a \le 0.0 \text{ then discard} $$
3. **Normalize Color:** Compute the final base color by dividing the weighted color RGB by the weight [S1, unofficial].
   $$ C_{\text{base}} = \frac{C_{\text{weighted}}.rgb}{C_{\text{weighted}}.a} $$
4. **Sample and Normalize Normal:** Read the normal from the normal G-buffer and normalize it [S1, unofficial].
   $$ N = \text{normalize}(\text{texture}(g\_normals, uv).xyz) $$
5. **Normalize Light Direction:** Ensure the light direction vector is normalized [S1, unofficial].
   $$ L = \text{normalize}(light\_dir\_view\_spc) $$
6. **Calculate Ambient Lighting:**
   $$ C_{\text{ambient}} = \text{ambient\_strength} \times \text{light\_color} $$
7. **Calculate Diffuse Lighting:**
   $$ C_{\text{diffuse}} = \max(-\text{dot}(N, L), 0.0) \times \text{light\_color} $$
   *Note: The negative sign in the dot product assumes $L$ points from the surface to the light source (view-space direction) [S1, unofficial].*
8. **Calculate Specular Lighting:**
   - Compute the reflection vector of the light direction about the normal [S1, unofficial].
     $$ R = \text{reflect}(L, N) $$
   - In view space, the view direction is typically $(0, 0, -1)$. The specular effect is calculated using the Z-component of the reflection vector (assuming standard view space where camera looks down -Z) [S1, unofficial].
     $$ S_{\text{effect}} = \max(R.z, 0.0)^{32} $$
   - Compute specular color [S1, unofficial].
     $$ C_{\text{specular}} = \text{specular\_strength} \times S_{\text{effect}} \times \text{light\_color} $$
9. **Combine Lighting:**
   $$ C_{\text{final}} = (C_{\text{ambient}} + C_{\text{diffuse}} + C_{\text{specular}}) \times C_{\text{base}} $$
10. **Output:** Write the final color to the framebuffer [S1, unofficial].
    $$ \text{FragColor} = \text{vec4}(C_{\text{final}}, 1.0) $$

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

**Parameters:**
- `ambient_strength`: Default 0.5 [S1, unofficial].
- `specular_strength`: Default 0.2 [S1, unofficial].
- `light_color`: Default (1.0, 1.0, 1.0) [S1, unofficial].
- `shininess`: Fixed at 32 in the reference implementation [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Division by Zero:** If a pixel is not covered by any splat, the weight in `g_colors` is 0. Dividing by this weight results in NaN. The discard step (Step 2) prevents this [S1, unofficial].
- **Light Direction Convention:** The source code uses `max(-dot(normal, light_dir), 0.0)` for diffuse. This implies `light_dir` is the vector from the surface to the light. If the light direction is provided as light-to-surface, the sign must be adjusted [S1, unofficial].
- **View Space Assumption:** The specular calculation assumes the camera is at the origin looking down the negative Z-axis in view space. If the view matrix is not identity or the coordinate system differs, the view direction vector must be computed explicitly [S1, unofficial].

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
