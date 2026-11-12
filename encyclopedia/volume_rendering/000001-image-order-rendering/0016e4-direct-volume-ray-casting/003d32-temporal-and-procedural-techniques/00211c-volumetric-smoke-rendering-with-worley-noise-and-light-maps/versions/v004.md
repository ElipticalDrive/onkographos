# Volumetric Smoke Rendering with Worley Noise and Light Maps

Also called: Unity Volumetric Smoke, Volumetric Smoke Rendering

## Overview
A real-time volumetric rendering technique for smoke implemented in Unity, combining 3D Worley noise for density generation, ray marching with adaptive step sizes, and front-to-back color blending to simulate light scattering and god rays [S1, unofficial]. It distinguishes itself from standard procedural cloud rendering by integrating a precomputed light map for scattering effects and utilizing Compute Shaders for texture generation, allowing for dynamic smoke simulation and efficient rendering on GPU [S1, unofficial].

## How it works
The method consists of two main phases: texture generation via Compute Shaders and real-time rendering via a vertex/fragment shader pipeline.

### Inputs and Outputs
**Inputs:**
- Camera position and view direction.
- Volume bounding box (AABB).
- Light direction and color.
- Noise parameters (octaves, persistence, lacunarity, amplitude, size).
- Simulation parameters (if using Navier-Stokes solver).

**Outputs:**
- Rendered smoke image with alpha channel.

### Data Structures
**Compute Shader Buffers:**
- `StructuredBuffer<float2> feature_points_buffer_2D`: Buffer of 2D feature points for 2D Worley noise [S1, unofficial].
- `RWTexture2D<float> noise_texture_2D`: Read-write 2D texture for storing 2D Worley noise [S1, unofficial].
- `StructuredBuffer<float3> feature_points_buffer_3D`: Buffer of 3D feature points for 3D Worley noise [S1, unofficial].
- `RWTexture3D<float> noise_texture_3D`: Read-write 3D texture for storing 3D Worley noise [S1, unofficial].
- `RWTexture2D<float> slice_texture`: Read-write 2D texture for storing a slice of the 3D noise texture [S1, unofficial].
- `RWTexture3D<float> light_density_texture`: Read-write 3D texture for storing light density [S1, unofficial].

**Merged Texture:**
- `union_texture`: RenderTexture with format `R8G8B8A8_UNorm`, dimension `Tex3D`, size `max_size x max_size x max_size` (where `max_size` is the max of `worley_noise_size`, `light_map_resolution`, and `solver.density_rt` dimensions). `enableRandomWrite` is true [S1, unofficial].

**MergedSample Struct:**
- Contains four floats: `noise`, `detail`, `light`, `mask` [S1, unofficial].

### Steps

#### 1. Worley Noise Generation
Worley noise is generated using Compute Shaders. The process involves iterating over neighboring cells to find the minimum distance to feature points.

**2D Worley Noise Kernel (`add_worley_noise_2D`):**
1.  Initialize `min_distance = 1.0`.
2.  Iterate over `offset_y` from -1 to 1 and `offset_x` from -1 to 1.
3.  Calculate `neighbour_cell_coordinates` and wrap them using `fmod`.
4.  Calculate `cellIndex` from wrapped coordinates.
5.  Calculate `featurePointOffset` as `neighbour_cell_coordinates + feature_points_buffer_2D[cellIndex]`.
6.  Update `min_distance = min(min_distance, distance(cell_coordinates + local_coordinates, featurePointOffset))`.
7.  Write `noise_texture_2D[id.xy] += (1.0 - min_distance) * noise_strength` [S1, unofficial].

**3D Worley Noise Kernel (`add_worley_noise_3D`):**
1.  Initialize `min_distance = 1.0`.
2.  Iterate over `offset_z`, `offset_y`, `offset_x` from -1 to 1.
3.  Calculate `neighbour_cell_coordinates` and wrap them using `fmod`.
4.  Calculate `cellIndex` from wrapped coordinates.
5.  Calculate `featurePointOffset` as `neighbour_cell_coordinates + feature_points_buffer_3D[cellIndex]`.
6.  Update `min_distance = min(min_distance, distance(cell_coordinates + local_coordinates, featurePointOffset))`.
7.  Write `noise_texture_3D[id.xyz] += (1.0 - min_distance) * noise_strength` [S1, unofficial].

**Octave Generation:**
1.  Loop from `octave = 0` to `octave_count - 1`.
2.  Set `worley_noise_division_size` to `(size, size, size)`.
3.  Call `_compute_worley_noise_3D_detail(amplitude)`.
4.  Update `amplitude *= persistence`.
5.  Update `size = Mathf.CeilToInt(size / lacunarity)` [S1, unofficial].

#### 2. Texture Merging
Textures are merged into a single `union_texture` for efficient sampling.

1.  Find kernel `unite_textures`.
2.  Set uniforms: `united_texture_size`, `noise_texture_size`, `detail_texture_size`, `light_texture_size`, `mask_texture_size`.
3.  Bind textures: `united_texture`, `noise_texture_merge`, `detail_texture_merge`, `light_texture_merge`, `mask_texture_merge`.
4.  Dispatch kernel with thread groups `ceil(max_size/8)` [S1, unofficial].

#### 3. Ray Marching
The fragment shader performs ray marching through the volume.

1.  **Vertex Shader:** Compute world position, screen position, and camera view direction for each vertex of the rendering quad/plane [S1, unofficial].
2.  **Fragment Shader:** Calculate local camera position and normalized view direction relative to the volume object [S1, unofficial].
3.  **Ray-AABB Intersection:** Compute entry point (`hit.x`) and distance inside the box (`hit.y`) for the ray originating at the camera [S1, unofficial].
    ```python
    def RayIntersectAABB(boundsMin, boundsMax, rayOrigin, rayDir):
        invRayDir = 1.0 / rayDir
        t0 = (boundsMin - rayOrigin) * invRayDir
        t1 = (boundsMax - rayOrigin) * invRayDir
        tmin = min(t0, t1)
        tmax = max(t0, t1)
        dstA = max(max(tmin.x, tmin.y), tmin.z)
        dstB = min(tmax.x, min(tmax.y, tmax.z))
        dstToBox = max(0, dstA)
        dstInsideBox = max(0, dstB - dstToBox)
        return (dstToBox, dstInsideBox)
    ```
4.  **Ray Start Point:** Calculate as `camera position + view direction * entry distance` [S1, unofficial].
5.  **Sampling:** At each step, sample the merged texture to get noise, detail, and mask values.
    -   **Non-united path:**
        -   Calculate `minimalized_object_scale = object_scale.xyz / max(max(object_scale.x, object_scale.y), object_scale.z)`.
        -   Scale `texCoord` by `minimalized_object_scale`.
        -   Sample noise from `_DensityTexture` at `scaled texCoord + 0.5`.
        -   Sample detail from `_DensityDetailTexture` at `scaled texCoord + 0.5`.
        -   If `use_light_map`, sample light from `_LightTexture` at `scaled texCoord + 0.5`; else set light to `0.65f`.
        -   Unscale `texCoord`.
        -   Sample mask from `_MaskTexture` at `original texCoord + 0.5` [S1, unofficial].
    -   **United path:**
        -   Calculate `final texCoord = fmod(texCoord + 0.5 + texture_offset / noise_texture_size + 1, 1)`.
        -   Sample data from `_UnitedTexture`.
        -   Set `sample_data.noise = data.r`, `detail = data.g`, `light = data.b`, `mask = data.a * 32.0f` [S1, unofficial].
6.  **Density Calculation:** Process sampled values to calculate density. If density is below `density_threshold`, replace with `god_rays_strength` to allow light to pass through empty areas for god ray effects [S1, unofficial].
7.  **Light Boost:** If density is near `god_rays_strength` threshold and light intensity is high, boost light intensity by 1.5x [S1, unofficial].
8.  **Front-to-Back Blending:** Blend color using front-to-back blending:
    ```python
    def BlendFTB(color, newColor):
        color.rgb += (1.0 - color.a) * newColor.a * newColor.rgb
        color.a += (1.0 - color.a) * newColor.a
        return color
    ```
9.  **Early Exit:** Break loop if accumulated alpha reaches 0.96 [S1, unofficial].
10. **Final Output:** Return final surface color with alpha clamped to 1.0, and RGB channels divided by alpha [S1, unofficial].

#### 4. Light Map Update
If `use_light_map` is enabled, update the light map periodically.
1.  If `counter <= 0`, call `_compute_light_map()` and reset counter [S1, unofficial].

#### 5. 3D Slice Iteration
If `iterate_over_z` is enabled, iterate through Z slices.
1.  If `iterate_z_counter <= 0`, increment `displaying_3d_slice` modulo `worley_noise_size.z` and reset counter [S1, unofficial].

### Parameters
-   `density_threshold`: Threshold for density values; if density is below this, it is treated as empty/god ray area [S1, unofficial].
-   `god_rays_strength`: Constant density value assigned to empty areas to allow god rays to shine through [S1, unofficial].
-   `light_decay`: Exponential decay factor for light intensity over distance [S1, unofficial].
-   `density_sample_count`: Number of steps for density sampling in the ray march [S1, unofficial].
-   `light_sample_count`: Number of steps for light sampling [S1, unofficial].
-   `light_sample_per_density_sample`: Ratio of light samples to density samples [S1, unofficial].
-   `light_color`: RGB color of the light source [S1, unofficial].
-   `light_direction`: Direction vector of the light source [S1, unofficial].
-   `noise_strength`: Multiplier for the Worley noise output [S1, unofficial].
-   `global_light_direction`: Direction of the global light [S1, unofficial].
-   `global_light_color`: Color of the global light [S1, unofficial].
-   `god_rays_intensity`: Intensity of the god rays effect [S1, unofficial].
-   `step_size`: Step size for light sampling [S1, unofficial].
-   `density_threashold`: Threshold for density in light sampling [S1, unofficial].
-   `octave_count`: Number of octaves for Worley noise generation. Default: 5 [S1, unofficial].
-   `persistance`: Amplitude multiplier per octave. Default: 0.5f [S1, unofficial].
-   `lacurancy`: Frequency multiplier per octave. Default: 2.0f [S1, unofficial].
-   `amplitude`: Initial amplitude for Worley noise. Default: 0.075f [S1, unofficial].
-   `size`: Initial resolution for Worley noise texture. Default: 16 [S1, unofficial].
-   `use_united_texture`: Boolean flag to switch between merged texture sampling and individual texture sampling. Default: 0 [S1, unofficial].
-   `use_light_map`: Boolean flag to enable light map sampling. Default: 0 [S1, unofficial].
-   `texture_offset`: Vector3 parameter for noise animation offset. Calculated as `(0, (-time_ms * 0.003f * worley_noise_size.y % worley_noise_size.y), 0)` [S1, unofficial].
-   `noise_texture_size`: Vector3 parameter for Worley noise texture dimensions [S1, unofficial].
-   `light_texture_size`: Vector3 parameter for light map texture dimensions [S1, unofficial].
-   `mask_texture_size`: Vector3 parameter for density mask texture dimensions [S1, unofficial].
-   `object_scale`: Vector4 parameter for object scaling, used to normalize texture coordinates [S1, unofficial].
-   `iterate_z_every`: Integer parameter controlling how often to iterate through Z slices. Default: 1 [S1, unofficial].
-   `update_light_map_every`: Integer parameter controlling how often to update the light map. Default: 1 [S1, unofficial].
-   `time_ms`: Float parameter for time in milliseconds, used for animation. Range: 0 to 128 [S1, unofficial].

### Edge Cases and Pitfalls
-   **Texture Coordinate Wrapping:** The united texture sampling uses `fmod` with `+1` to handle negative coordinates and ensure wrapping within `[0,1)` [S1, unofficial].
-   **Mask Scaling:** The mask value from the united texture is scaled by `32.0f` [S1, unofficial].
-   **Light Map Fallback:** If `use_light_map` is false, the light value is hardcoded to `0.65f` [S1, unofficial].

## Operating characteristics
-   **Time Complexity:** Depends on the number of ray marching steps (`density_sample_count`) and the complexity of the Worley noise generation (number of octaves and grid size).
-   **Memory Complexity:** Proportional to the size of the 3D noise texture and light map texture.
-   **Performance:** Real-time performance is achieved by using Compute Shaders for preprocessing and optimizing the ray marching loop with early exit conditions [S1, unofficial].
-   **Strengths:** Efficient real-time rendering, dynamic smoke effects, support for god rays and light scattering [S1, unofficial].
-   **Limitations:** Quality depends on the resolution of the noise textures and the number of ray marching steps [S1, unofficial].

## Background
This technique is implemented in Unity projects for real-time volumetric smoke effects, leveraging Compute Shaders for preprocessing and a standard vertex/fragment shader for rendering [S1, unofficial]. It builds upon the principles of procedural volumetric rendering and ray marching.

## Sources
[S1] furkanunsal42/VolumetricSmoke, https://github.com/furkanunsal42/VolumetricSmoke, fetched 2026-10-26 (unofficial)
