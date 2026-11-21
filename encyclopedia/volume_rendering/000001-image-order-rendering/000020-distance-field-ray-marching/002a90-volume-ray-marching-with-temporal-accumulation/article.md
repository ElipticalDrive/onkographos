# Volume Ray Marching with Temporal Accumulation

Also called: HZL Fluid Effects Volume Rendering

## Overview
Volume Ray Marching with Temporal Accumulation is a real-time rendering technique that integrates the Radiative Transfer Equation (RTE) through a 3D density field using fixed-step ray marching, stabilized by blending the current frame's result with a history texture [S1, unofficial]. It distinguishes itself from standard ray marching by reducing temporal noise and flickering in animated sequences, making it suitable for fluid simulations where density fields change rapidly [S1, unofficial]. Unlike [SDF Cloud Rendering](2064-sdf-cloud-rendering.md) which relies on analytic distance fields, this method operates directly on sampled scalar textures, and unlike [Atmospheric Ray Marching](4874-atmospheric-ray-marching.md) which models global atmospheric layers, it targets localized volumetric effects like smoke and fire within a bounded box [S1, unofficial].

## How it works
The method renders a full-screen quad, computing a ray for each fragment that intersects a unit cube $[-1, 1]^3$. It samples a density texture, computes lighting (including optional self-shadowing), and accumulates color. The final output is temporally blended with a previous frame's result to smooth noise [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   `densityTexture`: 3D texture storing density values.
*   `sceneDepthTexture`: Depth buffer of the scene geometry to clip rays.
*   `historyTexture`: RGBA texture storing the accumulated color from the previous frame.
*   `cameraPosition`: World-space camera origin.
*   `inverseViewProjection`: Inverse of the combined view-projection matrix.
*   `lightDirection`: Normalized direction vector of the light source.
*   Parameters: `stepSize`, `maximumSteps`, `densityScale`, `absorption`, `emissionStrength`, `lightIntensity`, `historyWeight` [S1, unofficial].

**Outputs:**
*   `m_colorTexture`: Final rendered frame (GL_RGBA8) [S1, unofficial].

### Data Structures
The implementation uses OpenGL textures with specific formats and filtering:
*   `m_colorTexture`: `GL_RGBA8`, filter `GL_LINEAR`, wrap `GL_CLAMP_TO_EDGE` [S1, unofficial].
*   `m_depthTexture`: `GL_DEPTH_COMPONENT24`, filter `GL_NEAREST`, wrap `GL_CLAMP_TO_EDGE` [S1, unofficial].
*   `m_sceneDepthTexture`: `GL_DEPTH_COMPONENT24`, filter `GL_NEAREST`, wrap `GL_CLAMP_TO_EDGE` [S1, unofficial].
*   `m_historyTexture`: `GL_RGBA8`, filter `GL_LINEAR`, wrap `GL_CLAMP_TO_EDGE`. Cleared to zero at initialization [S1, unofficial].

### Numbered Steps

1.  **Initialize Textures**: Allocate the four textures described above. Set filtering modes as specified. Clear `m_historyTexture` to `(0, 0, 0, 0)` [S1, unofficial].
2.  **Compute Ray Origin and Direction**: Convert fragment texture coordinate to clip space. Unproject the far plane point using `inverseViewProjection` to get `farWorld`. Calculate `rayDirection = normalize(farWorld - cameraPosition)` [S1, unofficial].
3.  **Intersect Ray with Unit Cube**: Compute intersection of ray origin and inverse ray direction with box $[-1, 1]$. Return interval $[t_{min}, t_{max}]$. Initialize `distanceAlongRay = max(t_{min}, 0.0)` [S1, unofficial].
4.  **Clip Ray Against Scene Geometry**: Sample `sceneDepthTexture` at fragment coordinates. If `sceneDepth < 1.0`, unproject the scene point to world space, calculate `sceneDistance` along `rayDirection`, and update `t_{exit} = min(t_{exit}, sceneDistance)` [S1, unofficial].
5.  **Discard if No Intersection**: If `t_{exit} <= distanceAlongRay`, discard the fragment [S1, unofficial].
6.  **Apply Jitter**: If `jitterEnabled`, add a random offset to `distanceAlongRay` using `random(gl_FragCoord.xy) * stepSize` [S1, unofficial].
7.  **Initialize Accumulation**: Set `accumulated = vec4(0.0)` [S1, unofficial].
8.  **Ray Marching Loop**: Loop index from $0$ to `maximumSteps`-1. Break if `index >= maximumSteps` or `distanceAlongRay >= t_{exit}`. Calculate `position = cameraPosition + rayDirection * distanceAlongRay`. Sample `densityTexture` at `position * 0.5 + 0.5`. Get `fieldValue` (r channel). Set `density = fieldValue` [S1, unofficial].
9.  **Detect Volume Entry**: If `!foundVolume` and `density > 0.01`, set `representativePosition = position` and `foundVolume = true` [S1, unofficial].
10. **Compute Color**: Determine `smokeColor` and `emittedLight` based on `fieldMode`: Mode 0 samples temperature/fire textures, computes ambient $(0.12, 0.15, 0.20)$ and direct light $(1.0, 0.88, 0.72) \times \text{lightIntensity} \times \text{transmittance}$, mixes smoke by position.y, and computes flame emission. Mode 1 mixes smoke color $(0.3, 0.04, 0.01)$ to $(1.0, 0.65, 0.08)$ by density. Modes 2/3 remap density to $|density - 0.5| \times 2.0$ and color red/blue [S1, unofficial].
11. **Accumulate Color and Opacity**: Compute step alpha $\alpha = 1.0 - e^{-\text{density} \times \text{stepSize}}$. Blend color into `accumulated` using standard alpha compositing [G].
12. **Temporal Accumulation**: If `temporalEnabled`, blend the current frame's result with `historyTexture` using `historyWeight` [S1, unofficial].
13. **Light Transmittance (Self-Shadowing)**: If `selfShadowingEnabled`, march 20 steps in direction `normalize(lightDirection + vec3(0.0001))` with step size 0.08. Accumulate `opticalDepth = texture(densityTexture, position * 0.5 + 0.5).r * 0.08`. Break if `abs(position) > 1.0`. Return transmittance $T = e^{-\text{opticalDepth} \times \text{absorption} \times \text{densityScale} \times 3.0}$ [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_volume_ray_marching_with_temporal_accumulation(
    u, v, camera_position, inverse_view_projection,
    scene_depth_texture, density_texture, history_texture,
    light_direction, step_size, maximum_steps, density_scale,
    absorption, emission_strength, light_intensity, history_weight,
    jitter_enabled, frame_index, field_mode, self_shadowing_enabled
):
    def mat_vec_mul(M, v):
        return vec4(
            dot(M[0], v), dot(M[1], v), dot(M[2], v), dot(M[3], v)
        )

    # 1. Compute Ray
    clip_pos = vec4(u * 2.0 - 1.0, v * 2.0 - 1.0, 1.0, 1.0)
    far_world = mat_vec_mul(inverse_view_projection, clip_pos)
    far_world = vec3(far_world) / far_world.w
    ray_origin = camera_position
    ray_direction = normalize(far_world - ray_origin)

    # 2. Intersect with Unit Cube [-1, 1]
    inverse_direction = 1.0 / ray_direction
    first = (-1.0 - ray_origin) * inverse_direction
    second = (1.0 - ray_origin) * inverse_direction
    minimum = min(first, second)
    maximum = max(first, second)
    t_entry = max(minimum.x, minimum.y, minimum.z)
    t_exit = min(maximum.x, maximum.y, maximum.z)
    t_entry = max(t_entry, 0.0)

    # 3. Clip against Scene Geometry
    scene_depth = scene_depth_texture.sample(u, v)
    if scene_depth < 1.0:
        clip_scene = vec4(u * 2.0 - 1.0, v * 2.0 - 1.0, scene_depth * 2.0 - 1.0, 1.0)
        world_scene = mat_vec_mul(inverse_view_projection, clip_scene)
        world_scene = vec3(world_scene) / world_scene.w
        t_scene = dot(world_scene - ray_origin, ray_direction)
        t_exit = min(t_exit, t_scene)

    if t_exit <= t_entry:
        return vec4(0.0)

    # 4. Jitter
    t = t_entry
    if jitter_enabled:
        coord = vec2(u, v)
        r = fract(sin(dot(coord * 12.9898, vec2(78.233, 45.164))) * 43758.5453)
        r = fract(r + frame_index * 0.01)
        t = t + r * step_size

    # 5. Initialize Accumulation
    accumulated_color = vec4(0.0)
    found_volume = False

    # 6. Ray Marching Loop
    for i in range(maximum_steps):
        if t >= t_exit:
            break

        position = ray_origin + ray_direction * t
        tex_coord = clamp(position * 0.5 + 0.5, 0.0, 1.0)
        density = density_texture.sample(tex_coord).r

        if not found_volume and density > 0.01:
            found_volume = True

        # Compute Lighting
        light_transmittance = 1.0
        if self_shadowing_enabled:
            optical_depth = 0.0
            light_dir = normalize(light_direction + vec3(0.0001))
            for _ in range(20):
                shadow_pos = position + light_dir * 0.08
                if abs(shadow_pos.x) > 1.0 or abs(shadow_pos.y) > 1.0 or abs(shadow_pos.z) > 1.0:
                    break
                shadow_density = density_texture.sample(shadow_pos * 0.5 + 0.5).r
                optical_depth += shadow_density * 0.08
            light_transmittance = exp(-optical_depth * absorption * density_scale * 3.0)

        smoke_color = vec3(0.0)
        emitted_light = vec3(0.0)

        if field_mode == 0:
            ambient = vec3(0.12, 0.15, 0.20)
            direct = vec3(1.0, 0.88, 0.72) * light_intensity * light_transmittance
            smoke_color = mix(vec3(0.1), vec3(0.5), position.y) * density
            emitted_light = vec3(1.0, 0.5, 0.1) * density * emission_strength
        elif field_mode == 1:
            smoke_color = mix(vec3(0.3, 0.04, 0.01), vec3(1.0, 0.65, 0.08), density)
        else:
            density = abs(density - 0.5) * 2.0
            smoke_color = vec3(1.0, 0.0, 0.0) if density > 0.5 else vec3(0.0, 0.0, 1.0)

        # Accumulate [G]
        alpha = 1.0 - exp(-density * step_size)
        color_step = (ambient + direct + emitted_light) * alpha
        accumulated_color.rgb += color_step * (1.0 - accumulated_color.a)
        accumulated_color.a += alpha * (1.0 - accumulated_color.a)

        t += step_size

    # 7. Temporal Accumulation
    if history_weight > 0.0:
        history_color = history_texture.sample(u, v)
        final_color = history_weight * history_color + (1.0 - history_weight) * accumulated_color
    else:
        final_color = accumulated_color

    return final_color
```

### Parameters
*   `gridSpacing`: float, default 0.5. Used for debug visualization of the grid, not the rendering itself [S1, unofficial].
*   `densityScale`: float. Multiplier for density values in light transmittance calculation [S1, unofficial].
*   `stepSize`: float. Distance advanced per ray march step [S1, unofficial].
*   `maximumSteps`: int. Maximum number of ray march steps per ray [S1, unofficial].
*   `frameIndex`: int. Current frame number, used for jitter randomization [S1, unofficial].
*   `jitterEnabled`: bool. Adds random offset to starting distance [S1, unofficial].
*   `adaptiveSteppingEnabled`: bool. Uniform present, but logic not explicitly shown in the core loop snippet [S1, unofficial].
*   `fieldMode`: int. 0: Smoke/Fire, 1: Simple Smoke, 2/3: Abstract/Color-coded [S1, unofficial].
*   `lightDirection`: vec3. Light source direction [S1, unofficial].
*   `lightIntensity`: float. Light intensity multiplier [S1, unofficial].
*   `absorption`: float. Absorption coefficient for Beer-Lambert law [S1, unofficial].
*   `emissionStrength`: float. Multiplier for emitted light [S1, unofficial].
*   `selfShadowingEnabled`: bool. Enables shadow ray marching [S1, unofficial].
*   `temporalEnabled`: bool. Enables temporal blending [S1, unofficial].
*   `historyWeight`: float. Weight for previous frame's color [S1, unofficial].

### Edge Cases and Pitfalls
*   **Division by Zero**: `intersectBox` computes `1.0 / rayDirection`. If `rayDirection` is zero, this results in infinity/NaN. The implementation relies on the caller providing normalized directions [S1, unofficial].
*   **Light Direction Normalization**: A small epsilon (0.0001) is added to `lightDirection` before normalization to prevent issues if the light direction is exactly zero [S1, unofficial].
*   **Texture Coordinate Mapping**: Density texture is sampled at `position * 0.5 + 0.5`. Texture wrap mode is set to `GL_CLAMP_TO_EDGE` during initialization to handle out-of-bounds coordinates [S1, unofficial].
*   **Precision**: Accumulation of alpha and color can suffer from precision loss if `stepSize` is very small or `maximumSteps` is very large. Standard `float` precision is used [G].

## Operating characteristics
*   **Time Complexity**: $O(N \times M)$, where $N$ is the number of pixels and $M$ is `maximumSteps`. Self-shadowing adds a constant factor (20 steps) per ray if enabled [G].
*   **Memory Complexity**: $O(N)$ for the output and history textures. The density texture size depends on the simulation resolution [G].
*   **Measured Performance**: Not given in the sources.
*   **Strengths**: Reduces temporal noise, stable for animated fluids, relatively simple to implement [S1, unofficial].
*   **Limitations**: Ghosting artifacts if `historyWeight` is too high or camera moves too fast. Fixed step size may lead to aliasing or slow convergence in sparse regions. Self-shadowing is expensive [S1, unofficial].
*   **When to Use**: Real-time fluid simulations, smoke, fire, clouds where temporal stability is important [S1, unofficial].
*   **When Not to Use**: Static scenes where offline rendering is acceptable, or scenes requiring high-frequency spatial detail without temporal smoothing [S1, unofficial].

## Background
This method is derived from the HZL Fluid Effects Studio, an open-source fluid simulation and rendering tool [S1, unofficial]. It implements a standard volume ray marching algorithm with the specific addition of temporal accumulation to handle the noise inherent in real-time volumetric rendering. The technique builds on the principles of [Volumetric and Atmospheric Rendering](4968-volumetric-and-atmospheric-rendering.md), specifically the discretization of the RTE along a ray [S1, unofficial].

## Sources
[S1] pvasbob/HZL-fluidEffectsStudio, https://github.com/pvasbob/HZL-fluidEffectsStudio, fetched 2026-10-27 (unofficial)
