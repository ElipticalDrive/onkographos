# Screen-space Post-processing

Also called: Composite Shader, Compositing/Post-Processing Pipeline, Screen Post-Processing

## Overview
Screen-space post-processing is a full-screen rendering stage that applies aspect-ratio preserving UV covering, luminance-based bright passes, color grading, a 17-tap box blur, and 1-LSB dithering to a scene texture to produce a display-ready image [S1, unofficial]. It differs from direct volume compositing methods by operating entirely in screen space on the final display image rather than during ray traversal, allowing for complex aesthetic effects like bloom and vignette without modifying the volume rendering pipeline [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method takes a source scene texture (typically the result of the volume rendering or main scene pass) and a set of uniform parameters controlling the effects [S1, unofficial]. The output is a final composited image [S1, unofficial].

**Data Structures:**
The implementation relies on a specific uniform buffer layout and vertex attributes for a full-screen quad pass [S1, unofficial].

*   **Uniforms Struct:**
    *   `view_proj`: `mat4x4<f32>`
    *   `inv_view_proj`: `mat4x4<f32>`
    *   `camera_pos`: `vec3<f32>`
    *   `time`: `f32`
    *   `sun_position`: `vec3<f32>`
    *   `is_underwater`: `f32`
    *   `screen_size`: `vec2<f32>`
    *   `water_level`: `f32`
    *   `moon_position`: `vec3<f32>`
    *   `moon_intensity`: `f32`
    *   `wind_dir_x`: `f32`
    *   `wind_dir_z`: `f32`
    *   `wind_speed`: `f32`
    *   `rain_factor`: `f32`
    *   `sky_visibility`: `f32`
    *   `menu_blur`: `f32` (Clamped to `[0.0, 1.0]`)
    *   `_pad_uniforms`: `f32`

*   **Vertex Attributes:**
    *   `positions`: `array<vec2<f32>, 3>` = `{(-1.0, -1.0), (3.0, -1.0), (-1.0, 3.0)}`
    *   `uvs`: `array<vec2<f32>, 3>` = `{(0.0, 1.0), (2.0, 1.0), (0.0, -1.0)}`

**Steps:**

1.  **UV Covering:** Adjust UVs to preserve aspect ratio if the screen and texture dimensions differ [S1, unofficial].
    *   Compute `texture_aspect = texture_size.x / max(texture_size.y, 1.0)` and `screen_aspect = screen_size.x / max(screen_size.y, 1.0)` [S1, unofficial].
    *   If `screen_aspect > texture_aspect`: `covered.y = 0.5 + (uv.y - 0.5) * (texture_aspect / screen_aspect)` [S1, unofficial].
    *   Else: `covered.x = 0.5 + (uv.x - 0.5) * (screen_aspect / texture_aspect)` [S1, unofficial].

2.  **Luminance Calculation:** Compute the luminance of the input color [S1, unofficial].
    *   `luma = dot(color, vec3(0.2126, 0.7152, 0.0722))` [S1, unofficial].

3.  **Bright Pass:** Isolate bright regions for bloom [S1, unofficial].
    *   `weight = smoothstep(0.68, 0.96, luminance(color))` [S1, unofficial].
    *   `output = color * weight` [S1, unofficial].

4.  **Vignette:** Apply a radial darkening effect, softened by `menu_blur` [S1, unofficial].
    *   `centered = uv * 2.0 - 1.0` [S1, unofficial].
    *   `centered.x *= screen_size.x / max(screen_size.y, 1.0)` [S1, unofficial].
    *   `vignette = 1.0 - smoothstep(0.42, 1.82, dot(centered, centered)) * mix(0.045, 0.025, blur)` [S1, unofficial].
    *   `output = clamp(color * vignette, vec3(0.0), vec3(1.0))` [S1, unofficial].

5.  **Underwater Refraction (Conditional):** If `is_underwater > 0.5` and `blur < 0.5`, split RGB channels to simulate refraction [S1, unofficial].
    *   `radial = (sample_uv - vec2(0.5)) * 1.6` [S1, unofficial].
    *   `shift = radial * texel * (1.2 + 0.25 * sin(time * 0.8))` [S1, unofficial].
    *   `color.r = sample_scene(sample_uv + shift).r` [S1, unofficial].
    *   `color.b = sample_scene(sample_uv - shift).b` [S1, unofficial].
    *   `color = mix(color, color * vec3(0.83, 0.96, 1.08), 0.28)` [S1, unofficial].

6.  **Cross-Shaped Bloom:** Add a cross-shaped blur to bright regions [S1, unofficial].
    *   `bloom_radius = texel * 3.25` [S1, unofficial].
    *   `bloom = bright_pass(color) * 0.28` [S1, unofficial].
    *   Add 4 neighbors at `±bloom_radius.x/y` with weight `0.18` each [S1, unofficial].
    *   `color += bloom * mix(0.105, 0.025, blur)` [S1, unofficial].

7.  **17-Tap Box Blur:** Apply a wide box blur controlled by `blur` intensity [S1, unofficial].
    *   If `blur > 0.001`: [S1, unofficial].
        *   `blurred = color * 0.08` [S1, unofficial].
        *   Add neighbors at `texel * (±4,0), (0,±4)` with weight `0.08` [S1, unofficial].
        *   Add neighbors at `texel * (±5.5, ±5.5)` with weight `0.07` [S1, unofficial].
        *   Add neighbors at `texel * (±10,0), (0,±10)` with weight `0.05` [S1, unofficial].
        *   Add neighbors at `texel * (±14, ±14)` with weight `0.03` [S1, unofficial].
        *   `color = mix(color, blurred * 0.82, blur)` [S1, unofficial].

8.  **Dithering:** Add 1-LSB dithering to reduce banding [S1, unofficial].
    *   `pixel = floor(uv * max(screen_size, vec2(1.0)))` [S1, unofficial].
    *   `screen_hash = fract(sin(dot(pixel, vec2(12.9898, 78.233))) * 43758.5453)` [S1, unofficial].
    *   `dither = (screen_hash - 0.5) / 255.0` [S1, unofficial].
    *   `color = clamp(color + vec3(dither), vec3(0.0), vec3(1.0))` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def screen_space_post_processing(uv, scene_texture, uniforms):
    """
    Applies full-screen post-processing effects to a single pixel.
    uv: vec2 in [0, 1]
    scene_texture: Texture2D representing the input scene.
    uniforms: Struct containing screen_size, menu_blur, is_underwater, time, etc.
    """
    screen_size = uniforms.screen_size
    menu_blur = clamp(uniforms.menu_blur, 0.0, 1.0)
    is_underwater = uniforms.is_underwater
    time = uniforms.time
    
    # 1. UV Covering
    texture_aspect = scene_texture.width / max(scene_texture.height, 1.0)
    screen_aspect = screen_size.x / max(screen_size.y, 1.0)
    
    covered_uv = uv
    if screen_aspect > texture_aspect:
        covered_uv.y = 0.5 + (uv.y - 0.5) * (texture_aspect / screen_aspect)
    else:
        covered_uv.x = 0.5 + (uv.x - 0.5) * (screen_aspect / texture_aspect)
        
    # Sample scene color
    color = scene_texture.sample(covered_uv)
    
    # 2. Luminance
    luma = dot(color, vec3(0.2126, 0.7152, 0.0722))
    
    # 3. Bright Pass
    weight = smoothstep(0.68, 0.96, luma)
    bright_pass_color = color * weight
    
    # 4. Vignette
    centered = uv * 2.0 - 1.0
    centered.x *= screen_size.x / max(screen_size.y, 1.0)
    vignette_factor = 1.0 - smoothstep(0.42, 1.82, dot(centered, centered)) * mix(0.045, 0.025, menu_blur)
    color = clamp(color * vignette_factor, vec3(0.0), vec3(1.0))
    
    # 5. Underwater Refraction
    if is_underwater > 0.5 and menu_blur < 0.5:
        texel = vec2(1.0 / scene_texture.width, 1.0 / scene_texture.height)
        radial = (covered_uv - vec2(0.5)) * 1.6
        shift = radial * texel * (1.2 + 0.25 * sin(time * 0.8))
        
        r_val = scene_texture.sample(covered_uv + shift).r
        b_val = scene_texture.sample(covered_uv - shift).b
        
        color.r = r_val
        color.b = b_val
        color = mix(color, color * vec3(0.83, 0.96, 1.08), 0.28)
        
    # 6. Cross-Shaped Bloom
    texel = vec2(1.0 / scene_texture.width, 1.0 / scene_texture.height)
    bloom_radius = texel * 3.25
    
    # Sample 4 neighbors for cross shape
    bloom_sum = vec3(0.0)
    bloom_sum += scene_texture.sample(covered_uv + vec2(bloom_radius.x, 0.0))
    bloom_sum += scene_texture.sample(covered_uv - vec2(bloom_radius.x, 0.0))
    bloom_sum += scene_texture.sample(covered_uv + vec2(0.0, bloom_radius.y))
    bloom_sum += scene_texture.sample(covered_uv - vec2(0.0, bloom_radius.y))
    
    bloom = (bright_pass_color * 0.28) + (bloom_sum * 0.18)
    color += bloom * mix(0.105, 0.025, menu_blur)
    
    # 7. 17-Tap Box Blur
    if menu_blur > 0.001:
        blurred = color * 0.08
        
        # Neighbors at (±4,0), (0,±4) weight 0.08
        blurred += scene_texture.sample(covered_uv + vec2(texel.x * 4, 0.0)) * 0.08
        blurred += scene_texture.sample(covered_uv - vec2(texel.x * 4, 0.0)) * 0.08
        blurred += scene_texture.sample(covered_uv + vec2(0.0, texel.y * 4)) * 0.08
        blurred += scene_texture.sample(covered_uv - vec2(0.0, texel.y * 4)) * 0.08
        
        # Neighbors at (±5.5, ±5.5) weight 0.07
        blurred += scene_texture.sample(covered_uv + vec2(texel.x * 5.5, texel.y * 5.5)) * 0.07
        blurred += scene_texture.sample(covered_uv + vec2(texel.x * 5.5, -texel.y * 5.5)) * 0.07
        blurred += scene_texture.sample(covered_uv - vec2(texel.x * 5.5, texel.y * 5.5)) * 0.07
        blurred += scene_texture.sample(covered_uv - vec2(texel.x * 5.5, -texel.y * 5.5)) * 0.07
        
        # Neighbors at (±10,0), (0,±10) weight 0.05
        blurred += scene_texture.sample(covered_uv + vec2(texel.x * 10, 0.0)) * 0.05
        blurred += scene_texture.sample(covered_uv - vec2(texel.x * 10, 0.0)) * 0.05
        blurred += scene_texture.sample(covered_uv + vec2(0.0, texel.y * 10)) * 0.05
        blurred += scene_texture.sample(covered_uv - vec2(0.0, texel.y * 10)) * 0.05
        
        # Neighbors at (±14, ±14) weight 0.03
        blurred += scene_texture.sample(covered_uv + vec2(texel.x * 14, texel.y * 14)) * 0.03
        blurred += scene_texture.sample(covered_uv + vec2(texel.x * 14, -texel.y * 14)) * 0.03
        blurred += scene_texture.sample(covered_uv - vec2(texel.x * 14, texel.y * 14)) * 0.03
        blurred += scene_texture.sample(covered_uv - vec2(texel.x * 14, -texel.y * 14)) * 0.03
        
        color = mix(color, blurred * 0.82, menu_blur)
        
    # 8. Dithering
    pixel = floor(uv * max(screen_size, vec2(1.0)))
    screen_hash = fract(sin(dot(pixel, vec2(12.9898, 78.233))) * 43758.5453)
    dither = (screen_hash - 0.5) / 255.0
    color = clamp(color + vec3(dither), vec3(0.0), vec3(1.0))
    
    return color
```

**Parameters:**
*   `menu_blur`: `f32`, controls blur intensity and vignette softening. Clamped to `[0.0, 1.0]` in fragment shader [S1, unofficial].
*   `is_underwater`: `f32`, enables RGB channel splitting refraction when `> 0.5` and `blur < 0.5` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Aspect Ratio Mismatch:** If `screen_aspect` and `texture_aspect` differ significantly, the UV covering step ensures the image is letterboxed or pillarboxed without distortion [S1, unofficial].
*   **Underwater Condition:** The refraction effect is only applied if `is_underwater > 0.5` AND `blur < 0.5`, preventing visual artifacts when heavy blurring is already active [S1, unofficial].

## Subtopics

### [Cloud Compositing & Tone Mapping](3628-cloud-compositing-tone-mapping.md)
Cloud Compositing & Tone Mapping blends resolved volumetric cloud radiance and transmittance with sky and ground background environments, then applies ACES-like tone mapping and gamma encoding to produce a display-ready image. It differs from general HDR post-processing pipelines by integrating scene-specific background synthesis (sun disc, horizon gradients, checkerboard ground) directly into the compositing step, rather than treating the background as a pre-computed texture.

### [HDR Post-Processing Pipeline](4633-hdr-post-processing-pipeline.md)
A 16-bit floating-point HDR rendering pipeline for scene post-processing. It sets itself apart from sibling methods like [Cloud Compositing & Tone Mapping](3628-cloud-compositing-tone-mapping.md) by integrating bloom and chromatic distortion directly into the tone-mapping fragment shader rather than using separate library passes, and from [Bloom](4906-bloom.md) by handling the bloom compositing within the same pass as tone mapping.

### [Bloom](4906-bloom.md)
Bloom is a screen-space post-processing effect that enhances the perception of brightness by isolating luminous regions via a soft threshold, blurring them across a pyramid of scales, and adding the result back to the original image. It differs from direct compositing methods by operating entirely in screen space on the final display image rather than during ray traversal.

### [Tilt-Shift Post-Processing](6901-tilt-shift-post-processing.md)
Tilt-Shift Post-Processing is a screen-space effect that simulates the shallow depth-of-field of a miniature model by keeping a horizontal band of the frame sharp while applying increasing Gaussian blur to areas above and below it, accompanied by increased color saturation. It distinguishes itself from general blur effects by its spatially varying strength profile (sharp band with falloff) and its specific aesthetic goal of mimicking macro photography of large-scale scenes. It is typically applied after the main scene render but before UI elements to ensure menus and text remain crisp.

### Bilateral Upsampling and Dual-Source Blending
A post-processing step to upscale and blend volumetric lighting results with the scene, using bilateral upsampling for resolution enhancement and dual-source blending for attenuation [S2, unofficial]. It performs bilateral upsampling on the resolved accumulation buffer, particularly helpful at 1/4 resolution, then additively blends the resolved/filtered output with the scene. It uses dual-source blending to attenuate the result based on medium density and depth [S2, unofficial].

### [Frustum Raycasting and Blending](3465-frustum-raycasting-and-blending.md)
Frustum Raycasting and Blending is a screen-space post-processing technique that casts rays from the camera through each pixel to intersect a cloud volume bounding box, blending the resulting cloud color with the original scene color. It addresses the artifacts of standard alpha blending (incorrect transparency order) and alpha testing (hard edges) by performing a screen-space depth-difference check during the final compositing pass. It differs from [Cloud Compositing & Tone Mapping](3628-cloud-compositing-tone-mapping.md) by explicitly using ray-box intersection logic in the fragment shader rather than pre-composited textures, and from [HDR Post-Processing Pipeline](4633-hdr-post-processing-pipeline.md) by focusing on volumetric occlusion rather than luminance scaling.

## Operating characteristics
Not given in the sources.

## Background
The specific implementation described here originates from the "Minerust" project, a voxel-based game engine using wgpu [S1, unofficial].

## Sources
[S1] B4rtekk1/Minerust, https://github.com/B4rtekk1/Minerust, fetched 2026-10-27 (unofficial)
[S2] Volumetric Light Scattering in Fallout 4, https://www.scribd.com/document/748135006/Fast-Flexible-Physically-Based-Volumetric-Light-Scattering, fetched 2026-10-29 (unofficial)
