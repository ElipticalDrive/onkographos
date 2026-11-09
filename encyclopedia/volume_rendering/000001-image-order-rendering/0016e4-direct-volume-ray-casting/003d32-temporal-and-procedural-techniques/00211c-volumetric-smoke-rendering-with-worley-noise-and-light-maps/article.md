# Volumetric Smoke Rendering with Worley Noise and Light Maps

Also called: Unity Volumetric Smoke, VolumetricSmoke

## Overview
A real-time volumetric rendering technique for smoke that combines 3D Worley noise for density generation, ray marching with adaptive step sizes, and front-to-back color blending to simulate light scattering and god rays [S1, unofficial]. It differs from static volume rendering by using procedural noise for density and optional light maps for scattering, allowing for dynamic, animated smoke effects in Unity [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Camera position, view direction, object bounding box (AABB), light direction, light color, noise parameters (octaves, persistence, lacunarity), and texture buffers (Worley noise, light map, density mask).
- **Outputs:** Final rendered color and alpha for each pixel.

**Data Structures:**
- `MergedSample`: A struct containing four floats: `noise`, `detail`, `light`, `mask` [S1, unofficial].
- `RWTexture3D<float> noise_texture_3D`: Read-write 3D texture for storing 3D Worley noise [S1, unofficial].
- `RWTexture2D<float> noise_texture_2D`: Read-write 2D texture for storing 2D Worley noise [S1, unofficial].
- `RWTexture2D<float> slice_texture`: Read-write 2D texture for storing a slice of the 3D noise texture [S1, unofficial].
- `RWTexture3D<float> light_density_texture`: Read-write 3D texture for storing light density [S1, unofficial].
- `union_texture`: A `RenderTexture` with format `R8G8B8A8_UNorm`, dimension `Tex3D`, and size `max_size x max_size x max_size` (where `max_size` is the maximum of `worley_noise_size`, `light_map_resolution`, and `solver.density_rt` dimensions). `enableRandomWrite` is true [S1, unofficial].
- `StructuredBuffer<float2> feature_points_buffer_2D`: Buffer of 2D feature points for 2D Worley noise [S1, unofficial].
- `StructuredBuffer<float3> feature_points_buffer_3D`: Buffer of 3D feature points for 3D Worley noise [S1, unofficial].

**Parameters:**
- `density_threshold`: Threshold for density values; if density is below this, it is treated as empty/god ray area [S1, unofficial].
- `god_rays_strength`: Constant density value assigned to empty areas to allow god rays to shine through [S1, unofficial].
- `light_decay`: Exponential decay factor for light intensity over distance [S1, unofficial].
- `density_sample_count`: Number of steps for density sampling in the ray march [S1, unofficial].
- `light_sample_count`: Number of steps for light sampling [S1, unofficial].
- `light_sample_per_density_sample`: Ratio of light samples to density samples [S1, unofficial].
- `light_color`: RGB color of the light source [S1, unofficial].
- `light_direction`: Direction vector of the light source [S1, unofficial].
- `noise_strength`: Multiplier for the Worley noise output [S1, unofficial].
- `global_light_direction`: Direction of the global light [S1, unofficial].
- `global_light_color`: Color of the global light [S1, unofficial].
- `god_rays_intensity`: Intensity of the god rays effect [S1, unofficial].
- `step_size`: Step size for light sampling [S1, unofficial].
- `density_threashold`: Threshold for density in light sampling [S1, unofficial].
- `octave_count`: Number of octaves for Worley noise generation. Default: 5 [S1, unofficial].
- `persistance`: Amplitude multiplier per octave in Worley noise generation. Default: 0.5f [S1, unofficial].
- `lacurancy`: Frequency multiplier per octave in Worley noise generation. Default: 2.0f [S1, unofficial].
- `amplitude`: Initial amplitude for Worley noise. Default: 0.075f [S1, unofficial].
- `size`: Initial resolution for Worley noise texture. Default: 16 [S1, unofficial].
- `use_united_texture`: Boolean flag (int) to switch between merged texture sampling and individual texture sampling. Default: 0 [S1, unofficial].
- `use_light_map`: Boolean flag (int) to enable light map sampling. Default: 0 [S1, unofficial].
- `texture_offset`: Vector3 parameter for noise animation offset. Calculated as `(0, (-time_ms * 0.003f * worley_noise_size.y % worley_noise_size.y), 0)` [S1, unofficial].
- `noise_texture_size`: Vector3 parameter for Worley noise texture dimensions [S1, unofficial].
- `light_texture_size`: Vector3 parameter for light map texture dimensions [S1, unofficial].
- `mask_texture_size`: Vector3 parameter for density mask texture dimensions [S1, unofficial].
- `object_scale`: Vector4 parameter for object scaling, used to normalize texture coordinates [S1, unofficial].
- `iterate_z_every`: Integer parameter controlling how often to iterate through Z slices for 3D slice display. Default: 1 [S1, unofficial].
- `update_light_map_every`: Integer parameter controlling how often to update the light map. Default: 1 [S1, unofficial].
- `time_ms`: Float parameter for time in milliseconds, used for animation. Range: 0 to 128 [S1, unofficial].

**Steps:**

1.  **Worley Noise Generation (Compute Shader):**
    -   **2D Noise:** For each pixel `(id.x, id.y)` in `noise_texture_2D`, calculate cell coordinates and local coordinates. Iterate over a 3x3 grid of neighboring cells. For each neighbor, find the wrapped cell index and retrieve the feature point from `feature_points_buffer_2D`. Calculate the minimum distance from the current texture coordinate to the feature point. Update the noise value: `noise_texture_2D[id.xy] += (1.0 - min_distance) * noise_strength` [S1, unofficial].
    -   **3D Noise:** Similar to 2D, but iterate over a 3x3x3 grid of neighboring cells in 3D space, using `feature_points_buffer_3D` and `noise_texture_3D` [S1, unofficial].
    -   **Octaves:** Loop from `octave 0` to `octave_count-1`. Set `worley_noise_division_size` to `(size, size, size)`. Call `_compute_worley_noise_3D_detail(amplitude)`. Update `amplitude *= persistance`. Update `size = Mathf.CeilToInt(size / lacurancy)` [S1, unofficial].

2.  **Texture Merging (Compute Shader):**
    -   Find kernel `unite_textures`. Set uniforms: `united_texture_size`, `noise_texture_size`, `detail_texture_size`, `light_texture_size`, `mask_texture_size`. Bind textures: `united_texture`, `noise_texture_merge`, `detail_texture_merge`, `light_texture_merge`, `mask_texture_merge`. Dispatch kernel with thread groups `ceil(max_size/8)` [S1, unofficial].

3.  **Slice Extraction (Compute Shader):**
    -   Find kernel `get_slice_of_3D_texture`. Set uniforms: `noise_texture_size_x`, `noise_texture_size_y`, `noise_texture_size_z`, `slice_index`. Bind textures: `noise_texture_3D`, `slice_texture`. Dispatch kernel with thread groups `ceil(noise_texture_size.x/8)`, `ceil(noise_texture_size.y/8)`, 1. Blit `slice_texture` to screen [S1, unofficial].

4.  **Ray Casting (Fragment Shader):**
    -   **Vertex Shader:** Compute world position, screen position, and camera view direction for each vertex of the rendering quad/plane [S1, unofficial].
    -   **Fragment Shader:** Calculate local camera position and normalized view direction relative to the volume object [S1, unofficial].
    -   **Ray-AABB Intersection:** Compute entry point (`hit.x`) and distance inside the box (`hit.y`) for the ray originating at the camera. The ray start point is `camera_position + view_direction * hit.x` [S1, unofficial].
    -   **Ray Marching Loop:**
        -   For each step, calculate the sample point.
        -   **Sampling:**
            -   **Non-united path:** If `!use_united_texture`, calculate `minimalized_object_scale = object_scale.xyz / max(max(object_scale.x, object_scale.y), object_scale.z)`. Scale `texCoord` by `minimalized_object_scale`. Sample noise from `_DensityTexture` at `scaled texCoord + 0.5`. Sample detail from `_DensityDetailTexture` at `scaled texCoord + 0.5`. If `use_light_map`, sample light from `_LightTexture` at `scaled texCoord + 0.5`; else set light to `0.65f`. Unscale `texCoord`. Sample mask from `_MaskTexture` at `original texCoord + 0.5` [S1, unofficial].
            -   **United path:** If `use_united_texture`, calculate `final texCoord = fmod(texCoord + 0.5 + texture_offset / noise_texture_size + 1, 1)`. Sample data from `_UnitedTexture`. Set `sample_data.noise = data.r`, `detail = data.g`, `light = data.b`, `mask = data.a * 32.0f` [S1, unofficial].
            -   **White Noise:** Calculate `texCoord = fmod(float2(texCoord.x + texCoord.y, texCoord.x - texCoord.y), 1)`. Sample from `_WhiteNoise` and return `r` component [S1, unofficial].
        -   **Density Calculation:** Process noise, detail, and mask values to calculate density. If density is below `density_threshold`, replace it with `god_rays_strength` [S1, unofficial]. If density is near `god_rays_strength` and light intensity is high, boost light intensity by 1.5x [S1, unofficial].
        -   **Light Sampling:** Sample light intensity based on `light_sample_count` and `light_decay` [S1, unofficial].
        -   **Front-to-Back Blending:** Blend the new color contribution: `new_color = light_intensity * density * color`. `accumulated_color.rgb += (1.0 - accumulated_color.a) * new_color.a * new_color.rgb`. `accumulated_color.a += (1.0 - accumulated_color.a) * new_color.a` [S1, unofficial].
        -   **Early Exit:** Break the loop if `accumulated_alpha >= 0.96` [S1, unofficial].
    -   **Final Output:** Return `accumulated_color` with alpha clamped to 1.0. Divide RGB channels by alpha (premultiplied to straight alpha conversion) [S1, unofficial].

5.  **Update Loop:**
    -   Set `use_united_texture` and `use_light_map`. If `iterate_time`, call `solver.step(Time.deltaTime * 1000)`. In `FixedUpdate`, set material textures and parameters. If `use_light_map` and `counter <= 0`, call `_compute_light_map()` and reset `counter`. If `iterate_z_counter <= 0`, if `iterate_over_z`, increment `displaying_3d_slice` modulo `worley_noise_size.z` and reset `counter` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Ray-AABB Intersection
def ray_intersect_aabb(bounds_min, bounds_max, ray_origin, ray_dir):
    inv_ray_dir = 1.0 / ray_dir
    t0 = (bounds_min - ray_origin) * inv_ray_dir
    t1 = (bounds_max - ray_origin) * inv_ray_dir
    tmin = min(t0, t1)
    tmax = max(t0, t1)
    dst_a = max(max(tmin.x, tmin.y), tmin.z)
    dst_b = min(tmax.x, min(tmax.y, tmax.z))
    dst_to_box = max(0, dst_a)
    dst_inside_box = max(0, dst_b - dst_to_box)
    return (dst_to_box, dst_inside_box)

# Front-to-Back Blending
def blend_ftb(color, new_color):
    color.rgb += (1.0 - color.a) * new_color.a * new_color.rgb
    color.a += (1.0 - color.a) * new_color.a
    return color

# Worley Noise 2D (Compute Shader)
def add_worley_noise_2d(id, noise_texture_size_x, noise_texture_size_y, cell_size_x, cell_size_y, noise_strength, feature_points_buffer_2d, noise_texture_2d):
    if id.x >= noise_texture_size_x or id.y >= noise_texture_size_y:
        return
    cell_count_x = noise_texture_size_x / cell_size_x
    cell_count_y = noise_texture_size_y / cell_size_y
    texture_coordinates = (id.x, id.y)
    cell_coordinates = floor(texture_coordinates / (cell_size_x, cell_size_y))
    local_coordinates = texture_coordinates / (cell_size_x, cell_size_y) - cell_coordinates
    min_distance = 1.0
    for offset_y in range(-1, 2):
        for offset_x in range(-1, 2):
            offset = (offset_x, offset_y)
            neighbour_cell_coordinates = cell_coordinates + offset
            wrapped_cell_coordinates = fmod(neighbour_cell_coordinates + (cell_count_x, cell_count_y), (cell_count_x, cell_count_y))
            cell_index = wrapped_cell_coordinates.x + wrapped_cell_coordinates.y * cell_count_x
            feature_point_offset = neighbour_cell_coordinates + feature_points_buffer_2d[cell_index]
            min_distance = min(min_distance, distance(cell_coordinates + local_coordinates, feature_point_offset))
    noise_texture_2d[id.xy] += (1.0 - min_distance) * noise_strength

# Worley Noise 3D (Compute Shader)
def add_worley_noise_3d(id, noise_texture_size_x, noise_texture_size_y, noise_texture_size_z, cell_size_x, cell_size_y, cell_size_z, noise_strength, feature_points_buffer_3d, noise_texture_3d):
    if id.x >= noise_texture_size_x or id.y >= noise_texture_size_y or id.z >= noise_texture_size_z:
        return
    cell_count_x = noise_texture_size_x / cell_size_x
    cell_count_y = noise_texture_size_y / cell_size_y
    cell_count_z = noise_texture_size_z / cell_size_z
    texture_coordinates = (id.x, id.y, id.z)
    cell_coordinates = floor(texture_coordinates / (cell_size_x, cell_size_y, cell_size_z))
    local_coordinates = texture_coordinates / (cell_size_x, cell_size_y, cell_size_z) - cell_coordinates
    min_distance = 1.0
    for offset_z in range(-1, 2):
        for offset_y in range(-1, 2):
            for offset_x in range(-1, 2):
                offset = (offset_x, offset_y, offset_z)
                neighbour_cell_coordinates = cell_coordinates + offset
                wrapped_cell_coordinates = fmod(neighbour_cell_coordinates + (cell_count_x, cell_count_y, cell_count_z), (cell_count_x, cell_count_y, cell_count_z))
                cell_index = wrapped_cell_coordinates.x + cell_count_x * (wrapped_cell_coordinates.y + cell_count_y * wrapped_cell_coordinates.z)
                feature_point_offset = neighbour_cell_coordinates + feature_points_buffer_3d[cell_index]
                min_distance = min(min_distance, distance(cell_coordinates + local_coordinates, feature_point_offset))
    noise_texture_3d[id.xyz] += (1.0 - min_distance) * noise_strength

# Ray Marching (Fragment Shader)
def ray_march(camera_position, view_direction, bounds_min, bounds_max, density_threshold, god_rays_strength, light_decay, density_sample_count, light_sample_count, light_sample_per_density_sample, light_color, light_direction, noise_strength, global_light_direction, global_light_color, god_rays_intensity, step_size, density_threashold, octave_count, persistance, lacurancy, amplitude, size, use_united_texture, use_light_map, texture_offset, noise_texture_size, light_texture_size, mask_texture_size, object_scale, iterate_z_every, update_light_map_every, time_ms):
    hit = ray_intersect_aabb(bounds_min, bounds_max, camera_position, view_direction)
    ray_start = camera_position + view_direction * hit.x
    ray_length = hit.y
    accumulated_color = (0.0, 0.0, 0.0, 0.0)
    
    for i in range(density_sample_count):
        t = i / density_sample_count
        sample_point = ray_start + view_direction * (t * ray_length)
        
        # Sampling
        if not use_united_texture:
            minimalized_object_scale = object_scale.xyz / max(max(object_scale.x, object_scale.y), object_scale.z)
            scaled_tex_coord = sample_point * minimalized_object_scale
            noise = sample_density_texture(scaled_tex_coord + 0.5)
            detail = sample_density_detail_texture(scaled_tex_coord + 0.5)
            light = sample_light_texture(scaled_tex_coord + 0.5) if use_light_map else 0.65
            unscaled_tex_coord = sample_point / minimalized_object_scale
            mask = sample_mask_texture(unscaled_tex_coord + 0.5)
        else:
            final_tex_coord = fmod(sample_point + 0.5 + texture_offset / noise_texture_size + 1, 1)
            data = sample_united_texture(final_tex_coord)
            noise = data.r
            detail = data.g
            light = data.b
            mask = data.a * 32.0
            
        # Density Calculation
        density = process_noise_detail_mask(noise, detail, mask)
        if density < density_threshold:
            density = god_rays_strength
        if abs(density - god_rays_strength) < epsilon and light > threshold:
            light *= 1.5
            
        # Light Sampling
        light_intensity = sample_light(sample_point, light_direction, light_color, light_decay, light_sample_count, light_sample_per_density_sample)
        
        # Blending
        new_color = light_intensity * density * global_light_color
        accumulated_color = blend_ftb(accumulated_color, new_color)
        
        if accumulated_color.a >= 0.96:
            break
            
    accumulated_color.a = min(accumulated_color.a, 1.0)
    if accumulated_color.a > 0:
        accumulated_color.rgb /= accumulated_color.a
    return accumulated_color
```

## Operating characteristics
- **Time Complexity:** O(N * M), where N is the number of pixels and M is the number of ray marching steps. The complexity of Worley noise generation is O(K * L^3), where K is the number of octaves and L is the texture size.
- **Memory Complexity:** O(L^3) for storing the 3D Worley noise texture and other related textures.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Performance scales with the number of ray marching steps and the resolution of the Worley noise texture.
- **Strengths:** Real-time rendering, procedural density generation, support for god rays and light scattering.
- **Limitations:** Limited to Unity engine, requires Compute Shaders for preprocessing, may suffer from aliasing if not properly filtered.
- **When to use:** Real-time volumetric smoke effects in Unity projects.
- **When not to use:** Offline rendering, non-Unity environments, static volumes.

## Background
This technique is used in Unity projects for real-time volumetric smoke effects, leveraging Compute Shaders for preprocessing and a standard vertex/fragment shader for rendering [S1, unofficial]. It is based on the principles of direct volume ray casting, with modifications for procedural noise and light map integration [S1, unofficial].

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
