# Screen-space Post-processing

Also called: Composite Shader, Compositing/Post-Processing Pipeline, Screen Post-Processing

## Overview
Screen-space post-processing is a full-screen rendering pass that applies aspect-ratio preserving UV covering, luminance-based bright passes, color grading, and spatial effects (such as bloom, blur, and vignetting) to a scene texture to produce a display-ready image [S1, unofficial]. It differs from ray-casting compositing methods by operating entirely in screen space on the final 2D image rather than during volumetric ray traversal, allowing for complex aesthetic effects like underwater refraction and tone mapping without modifying the volume rendering pipeline [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A scene texture (typically the result of the volume raycasting or scene rendering pass), uniform parameters (screen size, time, blur intensity, underwater flag), and vertex attributes for a full-screen quad.
*   **Output:** A final color buffer ready for display.

**Data Structures:**
*   **Uniforms:** A struct containing rendering and scene state.
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
1.  **UV Covering:** Compute `texture_aspect = texture_size.x / max(texture_size.y, 1.0)` and `screen_aspect = screen_size.x / max(screen_size.y, 1.0)`.
    *   If `screen_aspect > texture_aspect`: `covered.y = 0.5 + (uv.y - 0.5) * (texture_aspect / screen_aspect)`.
    *   Else: `covered.x = 0.5 + (uv.x - 0.5) * (screen_aspect / texture_aspect)`.
2.  **Luminance Calculation:** Compute `luma = dot(color, vec3(0.2126, 0.7152, 0.0722))`.
3.  **Bright Pass:** Compute `weight = smoothstep(0.68, 0.96, luminance(color))`. Output `bright_pass = color * weight`.
4.  **Vignette:** Compute `centered = uv * 2.0 - 1.0`. Adjust `centered.x *= screen_size.x / max(screen_size.y, 1.0)`. Compute `vignette = 1.0 - smoothstep(0.42, 1.82, dot(centered, centered)) * mix(0.045, 0.025, blur)`. Output `vignette_color = clamp(color * vignette, vec3(0.0), vec3(1.0))`.
5.  **Underwater Refraction (Conditional):** If `is_underwater > 0.5` and `blur < 0.5`:
    *   Compute `radial = (sample_uv - vec2(0.5)) * 1.6`.
    *   Compute `shift = radial * texel * (1.2 + 0.25 * sin(time * 0.8))`.
    *   Sample Red channel: `color.r = sample_scene(sample_uv + shift).r`.
    *   Sample Blue channel: `color.b = sample_scene(sample_uv - shift).b`.
    *   Mix: `color = mix(color, color * vec3(0.83, 0.96, 1.08), 0.28)`.
6.  **Cross-Shaped Bloom:**
    *   `bloom_radius = texel * 3.25`.
    *   `bloom = bright_pass(color) * 0.28`.
    *   Add 4 neighbors at `±bloom_radius.x/y` with weight `0.18` each.
    *   `color += bloom * mix(0.105, 0.025, blur)`.
7.  **17-Tap Box Blur (Conditional):** If `blur > 0.001`:
    *   Initialize `blurred = color * 0.08`.
    *   Add neighbors at `texel * (±4, 0), (0, ±4)` with weight `0.08`.
    *   Add neighbors at `texel * (±5.5, ±5.5)` with weight `0.07`.
    *   Add neighbors at `texel * (±10, 0), (0, ±10)` with weight `0.05`.
    *   Add neighbors at `texel * (±14, ±14)` with weight `0.03`.
    *   `color = mix(color, blurred * 0.82, blur)`.
8.  **Dithering:**
    *   `pixel = floor(uv * max(screen_size, vec2(1.0)))`.
    *   `screen_hash = fract(sin(dot(pixel, vec2(12.9898, 78.233))) * 43758.5453)`.
    *   `dither = (screen_hash - 0.5) / 255.0`.
    *   `color = clamp(color + vec3(dither), vec3(0.0), vec3(1.0))`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
# Inputs: vertex_index (0, 1, 2)
# Outputs: out_uv, out_position

def vertex_shader(vertex_index):
    positions = [(-1.0, -1.0), (3.0, -1.0), (-1.0, 3.0)]
    uvs = [(0.0, 1.0), (2.0, 1.0), (0.0, -1.0)]
    out_position = positions[vertex_index]
    out_uv = uvs[vertex_index]
    return out_position, out_uv

# Fragment Shader
# Inputs: in_uv, uniforms (screen_size, time, menu_blur, is_underwater, etc.)
# Outputs: out_color

def fragment_shader(in_uv):
    # 1. UV Covering
    texture_aspect = texture_size.x / max(texture_size.y, 1.0)
    screen_aspect = screen_size.x / max(screen_size.y, 1.0)
    
    covered = in_uv
    if screen_aspect > texture_aspect:
        covered.y = 0.5 + (in_uv.y - 0.5) * (texture_aspect / screen_aspect)
    else:
        covered.x = 0.5 + (in_uv.x - 0.5) * (screen_aspect / texture_aspect)
        
    # Sample scene color
    color = sample_scene(covered)
    
    # 2. Luminance
    luma = dot(color, vec3(0.2126, 0.7152, 0.0722))
    
    # 3. Bright Pass
    weight = smoothstep(0.68, 0.96, luma)
    bright_pass = color * weight
    
    # 4. Vignette
    centered = in_uv * 2.0 - 1.0
    centered.x *= screen_size.x / max(screen_size.y, 1.0)
    vignette_factor = 1.0 - smoothstep(0.42, 1.82, dot(centered, centered)) * mix(0.045, 0.025, menu_blur)
    color = clamp(color * vignette_factor, vec3(0.0), vec3(1.0))
    
    # 5. Underwater Refraction
    if is_underwater > 0.5 and menu_blur < 0.5:
        texel = 1.0 / screen_size
        radial = (covered - vec2(0.5)) * 1.6
        shift = radial * texel * (1.2 + 0.25 * sin(time * 0.8))
        
        r_sample = sample_scene(covered + shift).r
        b_sample = sample_scene(covered - shift).b
        
        color.r = r_sample
        color.b = b_sample
        color = mix(color, color * vec3(0.83, 0.96, 1.08), 0.28)
        
    # 6. Cross-Shaped Bloom
    texel = 1.0 / screen_size
    bloom_radius = texel * 3.25
    
    # Sample bright pass neighbors
    bloom_sum = bright_pass * 0.28
    bloom_sum += sample_bright_pass(covered + vec2(bloom_radius.x, 0.0)) * 0.18
    bloom_sum += sample_bright_pass(covered - vec2(bloom_radius.x, 0.0)) * 0.18
    bloom_sum += sample_bright_pass(covered + vec2(0.0, bloom_radius.y)) * 0.18
    bloom_sum += sample_bright_pass(covered - vec2(0.0, bloom_radius.y)) * 0.18
    
    color += bloom_sum * mix(0.105, 0.025, menu_blur)
    
    # 7. 17-Tap Box Blur
    if menu_blur > 0.001:
        blurred = color * 0.08
        # Neighbors at (±4,0), (0,±4)
        blurred += sample_scene(covered + vec2(texel.x * 4.0, 0.0)) * 0.08
        blurred += sample_scene(covered - vec2(texel.x * 4.0, 0.0)) * 0.08
        blurred += sample_scene(covered + vec2(0.0, texel.y * 4.0)) * 0.08
        blurred += sample_scene(covered - vec2(0.0, texel.y * 4.0)) * 0.08
        # Neighbors at (±5.5, ±5.5)
        blurred += sample_scene(covered + vec2(texel.x * 5.5, texel.y * 5.5)) * 0.07
        blurred += sample_scene(covered + vec2(texel.x * 5.5, -texel.y * 5.5)) * 0.07
        blurred += sample_scene(covered - vec2(texel.x * 5.5, texel.y * 5.5)) * 0.07
        blurred += sample_scene(covered - vec2(texel.x * 5.5, -texel.y * 5.5)) * 0.07
        # Neighbors at (±10,0), (0,±10)
        blurred += sample_scene(covered + vec2(texel.x * 10.0, 0.0)) * 0.05
        blurred += sample_scene(covered - vec2(texel.x * 10.0, 0.0)) * 0.05
        blurred += sample_scene(covered + vec2(0.0, texel.y * 10.0)) * 0.05
        blurred += sample_scene(covered - vec2(0.0, texel.y * 10.0)) * 0.05
        # Neighbors at (±14, ±14)
        blurred += sample_scene(covered + vec2(texel.x * 14.0, texel.y * 14.0)) * 0.03
        blurred += sample_scene(covered + vec2(texel.x * 14.0, -texel.y * 14.0)) * 0.03
        blurred += sample_scene(covered - vec2(texel.x * 14.0, texel.y * 14.0)) * 0.03
        blurred += sample_scene(covered - vec2(texel.x * 14.0, -texel.y * 14.0)) * 0.03
        
        color = mix(color, blurred * 0.82, menu_blur)
        
    # 8. Dithering
    pixel = floor(in_uv * max(screen_size, vec2(1.0)))
    screen_hash = fract(sin(dot(pixel, vec2(12.9898, 78.233))) * 43758.5453)
    dither = (screen_hash - 0.5) / 255.0
    color = clamp(color + vec3(dither), vec3(0.0), vec3(1.0))
    
    return color
```

**Parameters:**
*   `menu_blur`: `f32`, controls blur intensity and vignette softening. Clamped to `[0.0, 1.0]` in fragment shader [S1, unofficial].
*   `is_underwater`: `f32`, enables RGB channel splitting refraction when `> 0.5` and `blur < 0.5` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Aspect Ratio Mismatch:** The UV covering step ensures that the scene texture is not distorted when the screen aspect ratio differs from the texture aspect ratio, preventing stretching artifacts [S1, unofficial].
*   **Underwater Condition:** The refraction effect is only applied if `is_underwater > 0.5` and `blur < 0.5`, preventing visual conflicts with heavy blurring effects [S1, unofficial].
*   **Precision:** Dithering is applied at the 1-LSB level to reduce banding artifacts in the final output [S1, unofficial].

## Subtopics

### [Cloud Compositing & Tone Mapping](3628-cloud-compositing-tone-mapping.md)
Cloud Compositing & Tone Mapping blends resolved volumetric cloud radiance and transmittance with sky and ground background environments, then applies ACES-like tone mapping and gamma encoding to produce a display-ready image. It differs from general HDR post-processing pipelines by integrating scene-specific background synthesis (sun disc, horizon gradients, checkerboard ground) directly into the compositing step, rather than treating the background as a pre-computed texture. [Cloud Compositing & Tone Mapping](3628-cloud-compositing-tone-mapping.md)

### [HDR Post-Processing Pipeline](4633-hdr-post-processing-pipeline.md)
A 16-bit floating-point HDR rendering pipeline for scene post-processing. It sets itself apart from sibling methods like [Cloud Compositing & Tone Mapping](3628-cloud-compositing-tone-mapping.md) by integrating bloom and chromatic distortion directly into the tone-mapping fragment shader rather than using separate library passes, and from [Bloom](4906-bloom.md) by handling the bloom compositing within the same pass as tone mapping. [HDR Post-Processing Pipeline](4633-hdr-post-processing-pipeline.md)

### [Bloom](4906-bloom.md)
Bloom is a screen-space post-processing effect that enhances the perception of brightness by isolating luminous regions via a soft threshold, blurring them across a pyramid of scales, and adding the result back to the original image. It differs from direct compositing methods by operating entirely in screen space on the final display image rather than during ray traversal. [Bloom](4906-bloom.md)

### [Tilt-Shift Post-Processing](6901-tilt-shift-post-processing.md)
Tilt-Shift Post-Processing is a screen-space effect that simulates the shallow depth-of-field of a miniature model by keeping a horizontal band of the frame sharp while applying increasing Gaussian blur to areas above and below it, accompanied by increased color saturation. It distinguishes itself from general blur effects by its spatially varying strength profile (sharp band with falloff) and its specific aesthetic goal of mimicking macro photography of large-scale scenes. It is typically applied after the main scene render but before UI elements to ensure menus and text remain crisp. [Tilt-Shift Post-Processing](6901-tilt-shift-post-processing.md)

### Bilateral Upsampling and Dual-Source Blending
A post-processing step to upscale and blend volumetric lighting results with the scene, using bilateral upsampling for resolution enhancement and dual-source blending for attenuation [S2, unofficial]. It performs bilateral upsampling on the resolved accumulation buffer, particularly helpful at 1/4 resolution, then additively blends the resolved/filtered output with the scene, and uses dual-source blending to attenuate the result based on medium density and depth [S2, unofficial].

### Frustum Raycasting and Blending
A post-processing step that casts rays from the camera through each pixel, intersects them with a cloud volume bounding box, and blends the resulting cloud color with the original scene color [S3, unofficial]. It samples the original scene color, sets the ray origin to the camera world position, normalizes the ray direction, and calculates ray-box intersection. If an intersection is found, it marches through the cloud volume to get cloud color and returns a linear interpolation between the original and cloud color. If no intersection, it returns the original color. The C# script calculates frustum corners at the far clip plane and passes them to the shader [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** The complexity is dominated by the number of texture samples per pixel. The 17-tap box blur and cross-shaped bloom require a fixed number of samples (17 and 5 respectively) per pixel, making the time complexity $O(1)$ with respect to scene complexity but dependent on the resolution.
*   **Memory Complexity:** The method requires storage for the scene texture, the output buffer, and temporary buffers for bloom and blur passes. The memory usage is proportional to the screen resolution.
*   **Measured Performance:** Specific performance metrics are not provided in the sources. However, the use of screen-space techniques allows for efficient implementation on modern GPUs, leveraging hardware-accelerated texture sampling and blending operations [S1, S2, unofficial].
*   **Scaling:** The method scales with screen resolution. The blur and bloom radii are defined in texel space, ensuring consistent visual appearance across different resolutions.
*   **Strengths:** Allows for complex aesthetic effects (vignette, bloom, refraction) without modifying the volume rendering pipeline. Efficient for real-time applications.
*   **Limitations:** Limited to screen-space effects; cannot account for occlusion or lighting interactions that occur outside the visible screen area. The specific implementation details (e.g., blur taps, bloom weights) are hardcoded and may not be optimal for all scenes [S1, unofficial].

## Background
Screen-space post-processing is a common technique in real-time graphics, used to enhance the visual quality of rendered images. The specific implementation described here is part of the Minerust project, a voxel-based game engine [S1, unofficial]. The technique is also used in other contexts, such as volumetric light scattering in Fallout 4, where bilateral upsampling and dual-source blending are employed to enhance volumetric lighting results [S2, unofficial]. Frustum raycasting and blending is used in volumetric cloud rendering to integrate cloud volumes with the scene [S3, unofficial].

## Sources
[S1] B4rtekk1/Minerust, https://github.com/B4rtekk1/Minerust, fetched 2026-10-27 (unofficial)
[S2] Volumetric Light Scattering in Fallout 4, https://www.scribd.com/document/748135006/Fast-Flexible-Physically-Based-Volumetric-Light-Scattering, fetched 2026-10-29 (unofficial)
[S3] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
