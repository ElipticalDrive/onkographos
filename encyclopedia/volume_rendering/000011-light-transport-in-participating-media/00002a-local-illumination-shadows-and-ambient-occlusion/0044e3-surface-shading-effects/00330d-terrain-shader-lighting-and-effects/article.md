# Terrain Shader Lighting and Effects

Also called: Voxel Terrain Lighting, Chunk Mesh Lighting, Directional Face Shading

## Overview
This method provides real-time lighting for voxel-based terrain rendered as chunk meshes, combining baked vertex colors with dynamic global and local illumination [S1, unofficial]. It solves the problem of achieving visually rich terrain appearance in voxel engines by blending face-dependent directional shading, soft shadows from spherical occluders, and point light attenuation, while applying post-processing effects like grid overlays and mode-switch visual rings [S1, unofficial]. It differs from pure volume rendering methods by operating on discrete surface normals of meshed chunks rather than continuous volume gradients, and from simple ambient occlusion methods by including explicit directional face shading and soft shadowing calculations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Vertex Inputs:** `vertexPosition` (vec3), `vertexNormal` (vec3), `vertexColor` (vec4, where RGB is albedo and A is a mode-switch factor).
*   **Uniforms:** Model matrix `matModel`, MVP matrix `mvp`, `sunDirection` (vec3), `sunColor` (vec3), `ambientColor` (vec3), `cameraPosition` (vec3), `fogColor` (vec3), `fogStart` (float), `fogEnd` (float), `occluders` (array of vec4, xyz=center, w=radius), `pointLights` (array of vec4, xyz=position, w=range), `pointLightColors` (array of vec3), `waveCenter` (vec3), `waveRadius` (float), `waveWidth` (float), `waveStrength` (float), `waveColor` (vec3), `gridSpacing` (float), `gridStrength` (float), `worldShade` (float).
*   **Constants:** `MAX_OCCLUDERS = 8`, `MAX_POINT_LIGHTS = 8`.
*   **Output:** Fragment color (vec3).

**Data Structures:**
*   **Occluders:** Array of 8 `vec4` structures. Each `vec4` represents a spherical occluder: `xyz` is the center position, `w` is the radius.
*   **Point Lights:** Array of 8 `vec4` structures. Each `vec4` represents a point light: `xyz` is the position, `w` is the maximum range.
*   **Point Light Colors:** Array of 8 `vec3` structures. Each `vec3` is the color of the corresponding point light.

**Steps:**

1.  **Vertex Shader Processing:**
    *   Transform the vertex position to clip space: `gl_Position = mvp * vec4(vertexPosition, 1.0)`.
    *   Transform the vertex normal to world space: `fragNormal = mat3(matModel) * vertexNormal`.
    *   Pass the vertex position to world space: `fragPosition = vec3(matModel * vec4(vertexPosition, 1.0))`.
    *   Pass the vertex color: `fragColor = vertexColor`.

2.  **Directional Face Shading:**
    *   Compute a directional shade factor based on the surface normal `n` (normalized `fragNormal`).
    *   Formula: `shade = n.x^2 * 0.80 + n.z^2 * 0.72 + n.y^2 * (n.y > 0.0 ? 1.00 : 0.55)`.
    *   This blends face shades based on squared normal components, making top faces brighter (1.00) and bottom faces darker (0.55) [S1, unofficial].

3.  **Sun Visibility and Shadowing:**
    *   Initialize `sunVisibility = 1.0`.
    *   For each occluder in `occluders`:
        *   Compute `toSun = -sunDirection`.
        *   Compute `relative = occluder.xyz - fragPosition`.
        *   Compute `along = dot(relative, toSun)`.
        *   If `along <= 0.0`, the occluder is behind the surface; skip.
        *   Compute perpendicular distance: `perpendicular = length(relative - toSun * along)`.
        *   Apply soft edge shadowing: `shadow = smoothstep(occluder.w * 0.80, occluder.w * 1.20, perpendicular)`.
        *   Update `sunVisibility *= (1.0 - shadow)`.

4.  **Point Light Attenuation:**
    *   Initialize `pointLightSum = vec3(0.0)`.
    *   For each point light in `pointLights`:
        *   Compute `distance = length(light.xyz - fragPosition)`.
        *   If `distance >= light.w`, skip.
        *   Compute attenuation: `attenuation = (1.0 - distance / light.w)^2`.
        *   Compute light direction: `toLight = normalize(light.xyz - fragPosition)`.
        *   Compute Lambertian term: `lambert = max(dot(normalize(fragNormal), toLight), 0.0)`.
        *   Add to sum: `pointLightSum += pointLightColors[i] * lambert * attenuation`.

5.  **Final Color Composition:**
    *   Normalize `fragNormal`.
    *   Compute albedo: `albedo = fragColor.rgb * mix(vec3(1.0), vec3(shade), worldShade)`.
    *   Compute diffuse sunlight: `sunDiffuse = max(dot(normalize(fragNormal), -sunDirection), 0.0) * sunVisibility`.
    *   Combine lighting: `litColor = albedo * (ambientColor + sunDiffuse * sunColor) + pointLightSum`.
    *   Mix between lit and self-lit (vertex alpha): `finalColor = mix(litColor, albedo, fragColor.a)`.

6.  **Grid Overlay:**
    *   If `gridStrength > 0.0`:
        *   Compute grid coordinates: `p = fragPosition / gridSpacing`.
        *   Compute width: `w = max(fwidth(p), 1e-4)`.
        *   Compute grid mask: `g = abs(fract(p - 0.5) - 0.5) / w + abs(normalize(fragNormal)) * 10.0`.
        *   Compute line mask: `line = clamp(1.0 - min(min(g.x, g.y), g.z), 0.0, 1.0)`.
        *   Fade with distance: `near = 1.0 - smoothstep(6.0, 14.0, length(cameraPosition - fragPosition))`.
        *   Apply: `finalColor *= 1.0 - 0.16 * line * near * gridStrength`.

7.  **Mode-Switch Wave:**
    *   If `waveStrength > 0.0`:
        *   Compute distance: `d = length(fragPosition.xz - waveCenter.xz)`.
        *   Compute band: `band = 1.0 - smoothstep(0.0, waveWidth, abs(d - waveRadius))`.
        *   Compute rim lighting: `rim = pow(1.0 - max(dot(normalize(fragNormal), normalize(cameraPosition - fragPosition)), 0.0), 2.0)`.
        *   Compute wave color: `waveContrib = waveColor * band * waveStrength * (0.3 + 0.5 * rim)`.
        *   Mute outside terrain: `mute = smoothstep(waveRadius, waveRadius + waveWidth, d)`.
        *   Apply: `finalColor = mix(finalColor, dot(finalColor, vec3(0.299, 0.587, 0.114)), mute) + waveContrib`.

8.  **Exponential Fog:**
    *   Compute fog factor: `fog = smoothstep(fogStart, fogEnd, length(cameraPosition - fragPosition))`.
    *   Mix color with fog: `finalColor = mix(finalColor, fogColor, fog)`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def terrain_shader_lighting_effects(
    vertex_position, vertex_normal, vertex_color,
    mvp, mat_model, sun_direction, sun_color, ambient_color,
    camera_position, fog_color, fog_start, fog_end,
    occluders, point_lights, point_light_colors,
    wave_center, wave_radius, wave_width, wave_strength, wave_color,
    grid_spacing, grid_strength, world_shade
):
    # Vertex Shader
    gl_position = mvp * vec4(vertex_position, 1.0)
    frag_normal = mat3(mat_model) * vertex_normal
    frag_position = vec3(mat_model * vec4(vertex_position, 1.0))
    frag_color = vertex_color

    # Directional Face Shading
    n = normalize(frag_normal)
    shade = n.x**2 * 0.80 + n.z**2 * 0.72 + n.y**2 * (1.0 if n.y > 0.0 else 0.55)

    # Sun Visibility
    sun_visibility = 1.0
    to_sun = -sun_direction
    for occluder in occluders:
        relative = occluder.xyz - frag_position
        along = dot(relative, to_sun)
        if along <= 0.0:
            continue
        perpendicular = length(relative - to_sun * along)
        shadow = smoothstep(occluder.w * 0.80, occluder.w * 1.20, perpendicular)
        sun_visibility *= (1.0 - shadow)

    # Point Light Attenuation
    point_light_sum = vec3(0.0)
    for i, light in enumerate(point_lights):
        distance = length(light.xyz - frag_position)
        if distance >= light.w:
            continue
        attenuation = (1.0 - distance / light.w)**2
        to_light = normalize(light.xyz - frag_position)
        lambert = max(dot(n, to_light), 0.0)
        point_light_sum += point_light_colors[i] * lambert * attenuation

    # Final Color Composition
    albedo = frag_color.rgb * mix(vec3(1.0), vec3(shade), world_shade)
    sun_diffuse = max(dot(n, -sun_direction), 0.0) * sun_visibility
    lit_color = albedo * (ambient_color + sun_diffuse * sun_color) + point_light_sum
    final_color = mix(lit_color, albedo, frag_color.a)

    # Grid Overlay
    if grid_strength > 0.0:
        p = frag_position / grid_spacing
        w = max(fwidth(p), 1e-4)
        g = abs(fract(p - 0.5) - 0.5) / w + abs(n) * 10.0
        line = clamp(1.0 - min(min(g.x, g.y), g.z), 0.0, 1.0)
        eye_distance = length(camera_position - frag_position)
        near = 1.0 - smoothstep(6.0, 14.0, eye_distance)
        final_color *= 1.0 - 0.16 * line * near * grid_strength

    # Mode-Switch Wave
    if wave_strength > 0.0:
        d = length(frag_position.xz - wave_center.xz)
        band = 1.0 - smoothstep(0.0, wave_width, abs(d - wave_radius))
        to_eye = normalize(camera_position - frag_position)
        rim = pow(1.0 - max(dot(n, to_eye), 0.0), 2.0)
        wave_contrib = wave_color * band * wave_strength * (0.3 + 0.5 * rim)
        mute = smoothstep(wave_radius, wave_radius + wave_width, d)
        luminance = dot(final_color, vec3(0.299, 0.587, 0.114))
        final_color = mix(final_color, luminance, mute) + wave_contrib

    # Exponential Fog
    eye_distance = length(camera_position - frag_position)
    fog = smoothstep(fog_start, fog_end, eye_distance)
    final_color = mix(final_color, fog_color, fog)

    return final_color
```

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of occluders and point lights. Each occluder and point light requires a constant number of vector operations per fragment.
*   **Memory Complexity:** Constant, determined by the maximum number of occluders and point lights (8 each).
*   **Strengths:** Provides visually appealing terrain lighting with soft shadows and face-dependent shading. Efficient for real-time rendering on chunk meshes.
*   **Limitations:** Limited to spherical occluders and point lights. Grid and wave effects are purely visual and do not affect lighting physics. Performance degrades with a large number of dynamic lights or occluders.
*   **When to Use:** Suitable for voxel-based terrain rendering where visual fidelity is important but full global illumination is too expensive.
*   **When Not to Use:** Not suitable for physically accurate light transport or complex shadowing scenarios.

## Background
This method is implemented in the VoxelTerraformer project, an unofficial voxel engine [S1, unofficial]. It is designed to provide a rich visual experience for voxel terrain by combining various lighting and post-processing effects in a fragment shader. The method is specific to the VoxelTerraformer engine and is not a general-purpose technique found in standard graphics literature.

## Sources
[S1] GhostlyActive/VoxelTerraformer, https://github.com/GhostlyActive/VoxelTerraformer, fetched 2026-10-27 (unofficial)
