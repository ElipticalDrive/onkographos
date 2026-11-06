# Volumetric Light Shafts (Cone Raymarching)

Also called: Post-process Volumetric Lighting, WebGPU Volumetrics

## Overview
Volumetric Light Shafts (Cone Raymarching) is a post-processing technique that renders volumetric light shafts by raymarching through a cone-shaped volume originating from a light source, using Signed Distance Fields (SDFs) to define the volume shape and noise for density variation [S1, unofficial]. It differs from standard volumetric rendering by operating in screen space on a pre-computed scene texture, using a cone SDF to create directional light shafts that taper from the light source rather than using box or sphere primitives [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Scene color texture, depth texture, shadow map, blue noise texture, camera parameters (position, view/projection inverses), light parameters (position, direction, transform), and frame index.
*   **Output:** Final color buffer with volumetric light shafts composited over the scene.

**Data Structures:**
*   **Constants Uniform Buffer:** Contains `projectionInverse` (mat4x4<f32>), `viewMatrixInverse` (mat4x4<f32>), `lightTransform` (mat4x4<f32>), `cameraPosition` (vec3f), `time` (f32), `lightPosition` (vec3f), `frameIndex` (u32), `lightDirection` (vec3f) [S1, unofficial].
*   **Texture Bindings:** Binding 0: Constants uniform; Binding 1: Scene texture (texture_2d<f32>); Binding 2: Depth texture (texture_depth_2d); Binding 3: Shadow texture (texture_depth_2d); Binding 4: Blue noise texture (texture_2d<f32>); Binding 5: Blue noise sampler [S1, unofficial].

**Parameters:**
*   `coneAngle`: Half-angle of the light cone in radians, set to `radians(30.0)` [S1, unofficial].
*   `stepSize`: Distance between raymarch steps, set to `0.5` [S1, unofficial].
*   `stepCount`: Maximum number of raymarch steps, set to `16` [S1, unofficial].
*   `fogDensity`: Base density multiplier, set to `0.03` [S1, unofficial].
*   `lightColor`: Color of the light, set to `vec3(1.0, 0.85, 0.85)` [S1, unofficial].
*   `grainIntensity`: Intensity of film grain, set to `1.0 / 8.0` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def volumetric_light_shafts(uv: vec2f, depth: f32, frag: vec2u, frame_index: u32, time: f32) -> vec4f:
    # 1. Reconstruct world position
    clip_z = depth * 2.0 - 1.0
    ndc = uv * 2.0 - 1.0
    clip = vec4f(ndc, clip_z, 1.0)
    world = constants.view_matrix_inverse * constants.projection_inverse * clip
    world_position = world.xyz / world.w

    # 2. Initialize raymarching
    ray_origin = constants.camera_position
    ray_direction = normalize(world_position - ray_origin)
    light_position = constants.light_position
    light_direction = constants.light_direction
    cone_angle = radians(30.0)
    cone_angle_half = cone_angle * 0.5
    step_size = 0.5
    step_count = 16
    light_intensity = 1.0
    accumulated_light = vec3f(0.0)
    transmittance = 5.0
    scene_depth = length(world_position - constants.camera_position)

    # 3. Calculate blue noise offset for jittering
    blue_noise_uv = (frag.xy / 64.0) + f32(frame_index & 0xFFFF) * 0.61803399f
    blue_noise_offset = texture_sample(blue_noise_texture, blue_noise_sampler, blue_noise_uv).r
    t = step_size + blue_noise_offset

    # 4. Raymarch loop
    for i in range(step_count):
        sample_position = ray_origin + ray_direction * t

        # Ray termination: break if beyond scene depth or hard limit
        if t > scene_depth or t > 16.0:
            break

        # Shadow check
        shadow = calculate_shadow(sample_position)
        if shadow == 0.0:
            t += step_size
            continue

        # Calculate SDF value for cone at sample position
        sdf_value = sd_cone(sample_position, light_position, light_direction, cone_angle_half)

        # Calculate shape factor using FBM noise
        shape_factor = -sdf_value + (fbm3d(sample_position) * 2.0)
        
        # Skip if shape factor is zero to avoid invalid density
        if shape_factor == 0.0:
            t += step_size
            continue

        # Calculate attenuation and phase
        distance_to_light = length(sample_position - light_position)
        attenuation = exp(-0.24 * distance_to_light)
        sample_light_direction = normalize(sample_position - light_position)
        scatter_phase = hg_phase(dot(ray_direction, -sample_light_direction))

        # Calculate luminance
        luminance = light_color * light_intensity * attenuation * scatter_phase

        # Calculate step density
        fog_density = 0.03
        step_density = max(fog_density * shape_factor, 0.0)

        # Beer's Law transmittance
        absorption = 1.0
        step_transmittance = exp(-step_density * step_size * absorption)
        transmittance *= step_transmittance

        # Accumulate light
        accumulated_light += luminance * transmittance * step_density * step_size

        t += step_size

    # 5. Final compositing
    scene_color = texture_load(scene_texture, frag, 0).rgb
    film_grain = hash2i(vec2i(vec2u(frag) * vec2u(frame_index) * vec2u(49218, 9123))) * grain_intensity - (grain_intensity / 2.0)
    final_color = black_and_white(scene_color + accumulated_light) + film_grain
    
    return vec4f(final_color, 1.0)

def calculate_shadow(world_position: vec3f) -> f32:
    clip = constants.light_transform * vec4f(world_position, 1.0)
    ndc = clip.xyz / clip.w
    coord = ndc.xy * 0.5 + 0.5
    
    # Shadow map boundary handling: if UV outside [0,1], point is fully lit
    if any(coord > vec2f(1.0)) or any(coord < vec2f(0.0)):
        return 1.0
    
    pixel_location = vec2f(coord.x, 1.0 - coord.y) * vec2f(texture_dimensions(shadow_texture)) * (1023.0 / 1024.0)
    shadow_map_depth = texture_load(shadow_texture, vec2u(pixel_location), 0)
    return step(ndc.z, shadow_map_depth)

def sd_cone(p: vec3f, axis_origin: vec3f, axis_dir: vec3f, angle_rad: f32) -> f32:
    p_to_origin = p - axis_origin
    h = dot(p_to_origin, axis_dir)
    r = length(p_to_origin - axis_dir * h)
    c = cos(angle_rad)
    s = sin(angle_rad)
    
    dist_to_surface_line = r * c - h * s
    dist_to_apex_plane = -h
    
    if h < 0.0 and dist_to_surface_line > 0.0:
        return length(p_to_origin)
    
    boundary_dists = vec2f(dist_to_surface_line, dist_to_apex_plane)
    return length(max(boundary_dists, vec2f(0.0))) + min(max(boundary_dists.x, boundary_dists.y), 0.0)

def fbm3d(p: vec3f) -> f32:
    value = 0.0
    amplitude = 0.5
    st = p + vec3f(time * 0.25, time * 0.125, 0.0)
    for i in range(4):
        value += amplitude * noise3d(st)
        st *= 2.0
        amplitude *= 0.5
    return value

def noise3d(st: vec3f) -> f32:
    s = st + vec3f(1234.0)
    i0 = vec3i(floor(s))
    f = fract(s)
    
    n0 = hash3i(i0)
    n1 = hash3i(i0 + vec3i(1,0,0))
    n2 = hash3i(i0 + vec3i(0,1,0))
    n3 = hash3i(i0 + vec3i(1,1,0))
    n4 = hash3i(i0 + vec3i(0,0,1))
    n5 = hash3i(i0 + vec3i(1,0,1))
    n6 = hash3i(i0 + vec3i(0,1,1))
    n7 = hash3i(i0 + vec3i(1,1,1))
    
    x00 = mix(n0, n1, f.x)
    x10 = mix(n2, n3, f.x)
    x01 = mix(n4, n5, f.x)
    x11 = mix(n6, n7, f.x)
    
    y0 = mix(x00, x10, f.y)
    y1 = mix(x01, x11, f.y)
    return mix(y0, y1, f.z)

def hash3i(p: vec3i) -> f32:
    ux = u32(p.x)
    uy = u32(p.y)
    uz = u32(p.z)
    return pcg((ux * 0x27d4eb2du + 29u) ^ (uy * 0x165667b1u - 1u) ^ (uz * 0x9e3779b1u + 9u))

def pcg(state: u32) -> f32:
    h = pcg_hash_u32(state)
    result = f32(h >> 8) * (1.0 / 16777216.0)
    return result

def pcg_hash_u32(x: u32) -> u32:
    state = x * 747796405u + 2891336453u
    word = ((state >> ((state >> 28u) + 4u)) ^ state) * 277803737u
    return (word >> 22u) ^ word

def hg_phase(mu: f32) -> f32:
    g = 0.375
    gg = g * g
    denom = max(1.0 + gg - 2.0 * g * mu, 0.001)
    scatter = (1.0 - gg) / pow(denom, 1.5)
    return scatter

def black_and_white(c: vec3f) -> vec3f:
    l = dot(c, vec3f(0.299, 0.587, 0.114))
    return vec3f(l)
```

**Edge Cases and Pitfalls:**
*   **Shadow Map Boundaries:** If UV coordinates calculated from light space are outside [0,1], the point is considered fully lit (shadow = 1.0) to avoid artifacts at the edges of the shadow map [S1, unofficial].
*   **SDF Zero Check:** If `shapeFactor` is exactly 0.0, the step is skipped (`continue`) to avoid division by zero or invalid density calculations [S1, unofficial].
*   **Ray Termination:** The raymarch loop breaks if the step distance `t` exceeds the scene depth (distance to the screen pixel) or exceeds a hard limit of 16.0 units, preventing infinite loops or rendering behind the camera [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** O(N * M) where N is the number of pixels and M is the number of raymarch steps (16). Each step involves SDF evaluation, noise sampling, and shadow map lookup.
*   **Memory Complexity:** O(1) per pixel for the raymarching state, plus storage for the scene, depth, shadow, and blue noise textures.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales linearly with resolution and step count. The use of blue noise jittering helps reduce banding artifacts without increasing step count significantly.
*   **Strengths:** Real-time post-processing approach; directional light shafts that taper naturally; uses SDF for efficient volume shape definition; noise adds visual detail.
*   **Limitations:** Limited to post-processing (cannot interact with dynamic objects not in the scene texture); shadow map resolution limits quality; fixed step count may lead to artifacts in complex scenes; no global illumination.
*   **When to use:** Real-time applications requiring volumetric light effects with limited computational resources; when light sources are directional and cone-shaped.
*   **When not to use:** When global illumination or complex light interactions are required; when high-quality volumetric scattering is needed beyond simple shafts.

## Background
Volumetric Light Shafts (Cone Raymarching) is an implementation based on the WebGPU-Volumetrics-Demo by Ne0nWinds [S1, unofficial]. It demonstrates a modern approach to volumetric lighting using WebGPU, leveraging SDFs and noise for efficient real-time rendering.

## Sources
[S1] Ne0nWinds/WebGPU-Volumetrics-Demo, https://github.com/Ne0nWinds/WebGPU-Volumetrics-Demo, fetched 2026-10-27 (unofficial)
