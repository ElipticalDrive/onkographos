# Path Tracing Extension with MIS

Also called: voxel path tracing

## Overview
A path tracing kernel that extends rays by marching them through sparse voxels, shading intersections using albedo and normal, and generating new ray directions using Multiple Importance Sampling (MIS) between sun disk and cosine-weighted hemisphere sampling [S1, unofficial]. It differs from standard path tracing by decoupling the accumulation buffer from the pixel grid, enabling spatial averaging of radiance estimates across hash buckets [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input Ray Buffer:** `StructuredBuffer<path_tracer_ray>`. Each ray contains `origin_and_path_index` (float4, where xyz is the origin and w is the path index as a uint) and `direction` (float3) [S1, unofficial].
*   **Path State Buffer:** `RWStructuredBuffer<path_tracer_path_state>`. Stores `radiance` (float3) and `throughput_and_spatial_index` (float3) [S1, unofficial].
*   **Output Ray Buffer:** `RWStructuredBuffer<path_tracer_ray>`. Stores extended rays for the next pass [S1, unofficial].
*   **Voxel Data:** `Texture3D<uint> voxels`. Each texel is a uint index into the `palette_rgba` buffer [S1, unofficial].
*   **Palette:** `StructuredBuffer<uint> palette_rgba`. Each uint is a packed RGBA color. Albedo is unpacked via `unpack_albedo(palette_rgba[voxels.Load(...)].r).rgb` [S1, unofficial].
*   **Uniforms:** `sun_direction` (float3), `transmitted_sun_color` (float3), `bounce` (uint), `cos_theta_max` (float, derived from solar radius), `sky_radiance` function [S1, unofficial].

**Data Structures:**
*   **Ray:** `origin_and_path_index` (float4), `direction` (float3) [S1, unofficial].
*   **Path State:** `radiance` (float3), `throughput_and_spatial_index` (float3) [S1, unofficial].
*   **Voxel Cell:** Packed uint. Bits 0-29 store cell coordinates, bits 30-31 store axis index [S1, unofficial].

**Parameters:**
*   `VX_PI_F`: The value of Pi used in PDF calculation. Exact value not given, but used as divisor [S1, unofficial].
*   `VX_SKY_SOLAR_RADIUS_RAD`: Solar radius in radians. Defined as `1.5f * 0.0044506f` [S1, unofficial].

**Steps:**
1.  **Dispatch Check:** If `thread_id >= input_ray_count[0]`, return [S1, unofficial].
2.  **Load Ray:** `ray = input_ray_buffer[thread_id]`. Extract `ray_origin = ray.origin_and_path_index.xyz`, `ray_dir = ray.direction`, `path_index = asuint(ray.origin_and_path_index.w)` [S1, unofficial].
3.  **March Ray:** `packed_cell = sparse_ray_march(ray_origin, ray_dir)` [S1, unofficial].
4.  **Handle Miss:** If `packed_cell == VX_NO_CELL`, add sky radiance to path state: `state.radiance.xyz += state.throughput_and_spatial_index.xyz * sky_radiance(ray_dir, uniforms.transmitted_sun_color.rgb)`. Update state and return [S1, unofficial].
5.  **Unpack Intersection:** `cell = unpack_voxel_cell(packed_cell)`, `axis_index = packed_cell >> 30u`. Compute normal: `axis_mask = float3(axis_index == uint3(0,1,2))`, `normal = -sign(ray_dir) * axis_mask` [S1, unofficial].
6.  **Compute Intersection Position:** `entry_plane = float3(cell) + (float3)(ray_dir < 0.0)`. `distance = dot(entry_plane - ray_origin, axis_mask) / dot(ray_dir, axis_mask)`. `position = ray_origin + distance * ray_dir` [S1, unofficial].
7.  **Fetch Albedo:** `albedo = unpack_albedo(palette_rgba[voxels.Load(int4(cell, 0)).r]).rgb` [S1, unofficial].
8.  **Sample Direction:** `sample_index = sample_ordinals[path_index] - 1u`. `samples = halton_sample_3d(sample_index, uniforms.bounce, path_index)`. `u = samples.xy`. `sample_sun = samples.z < 0.5`. If `sample_sun`, `local_dir = sample_cone(u, cos_theta_max)`. Else `local_dir = sample_cosine_weighted_hemisphere(u)` [S1, unofficial].
9.  **Orient Direction:** If `sample_sun`, `next_ray_dir = orient_sample_direction(local_dir, uniforms.sun_direction.xyz)`. Else `next_ray_dir = orient_axis_aligned_sample_direction(local_dir, normal)` [S1, unofficial].
10. **Compute Shading:** `n_dot_l = max(dot(normal, next_ray_dir), 0.0)`. If `n_dot_l == 0.0`, return [S1, unofficial].
11. **Compute PDF:** `pdf = 0.5 * (pdf_cone(dot(uniforms.sun_direction.xyz, next_ray_dir), cos_theta_max) + pdf_cosine_weighted_hemisphere(n_dot_l))` [S1, unofficial].
12. **Update Throughput:** `throughput = state.throughput_and_spatial_index.xyz`. `throughput *= albedo * n_dot_l / (VX_PI_F * max(pdf, 1e-6))`. Update state [S1, unofficial].
13. **Output Ray:** `InterlockedAdd(output_ray_count[0], 1u, output_ray_index)`. `output_ray.origin_and_path_index = float4(offset_ray(position, normal), asfloat(path_index))`. `output_ray.direction = next_ray_dir`. Write to `output_ray_buffer[output_ray_index]` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Path Tracing Extension with MIS
# Assumes standard math operations (dot, cross, max, sign, etc.)
# and specific helper functions defined in the source context.

def path_tracer_extend(thread_id, input_ray_count, input_ray_buffer, 
                       output_ray_buffer, output_ray_count, state_buffer,
                       voxels, palette_rgba, uniforms):
    
    # 1. Dispatch Check
    if thread_id >= input_ray_count[0]:
        return

    # 2. Load Ray
    ray = input_ray_buffer[thread_id]
    ray_origin = ray.origin_and_path_index.xyz
    ray_dir = ray.direction
    path_index = asuint(ray.origin_and_path_index.w)

    # 3. March Ray
    packed_cell = sparse_ray_march(ray_origin, ray_dir)

    # 4. Handle Miss
    if packed_cell == VX_NO_CELL:
        state = state_buffer[thread_id] # Assuming state is indexed by thread or ray
        sky_rad = sky_radiance(ray_dir, uniforms.transmitted_sun_color.rgb)
        state.radiance.xyz += state.throughput_and_spatial_index.xyz * sky_rad
        state_buffer[thread_id] = state
        return

    # 5. Unpack Intersection
    cell = unpack_voxel_cell(packed_cell)
    axis_index = packed_cell >> 30u
    axis_mask = float3(axis_index == uint3(0, 1, 2))
    normal = -sign(ray_dir) * axis_mask

    # 6. Compute Intersection Position
    entry_plane = float3(cell) + (float3)(ray_dir < 0.0)
    denominator = dot(ray_dir, axis_mask)
    # Avoid division by zero if denominator is 0, though axis_mask ensures one component is 1 or -1
    distance = dot(entry_plane - ray_origin, axis_mask) / denominator
    position = ray_origin + distance * ray_dir

    # 7. Fetch Albedo
    voxel_index = voxels.Load(int4(cell, 0)).r
    albedo = unpack_albedo(palette_rgba[voxel_index].r).rgb

    # 8. Sample Direction
    sample_index = sample_ordinals[path_index] - 1u
    samples = halton_sample_3d(sample_index, uniforms.bounce, path_index)
    u = samples.xy
    sample_sun = samples.z < 0.5

    if sample_sun:
        local_dir = sample_cone(u, uniforms.cos_theta_max)
    else:
        local_dir = sample_cosine_weighted_hemisphere(u)

    # 9. Orient Direction
    if sample_sun:
        next_ray_dir = orient_sample_direction(local_dir, uniforms.sun_direction.xyz)
    else:
        next_ray_dir = orient_axis_aligned_sample_direction(local_dir, normal)

    # 10. Compute Shading
    n_dot_l = max(dot(normal, next_ray_dir), 0.0)
    if n_dot_l == 0.0:
        return

    # 11. Compute PDF
    pdf_cone_val = pdf_cone(dot(uniforms.sun_direction.xyz, next_ray_dir), uniforms.cos_theta_max)
    pdf_cosine_val = pdf_cosine_weighted_hemisphere(n_dot_l)
    pdf = 0.5 * (pdf_cone_val + pdf_cosine_val)

    # 12. Update Throughput
    state = state_buffer[thread_id] # Assuming state is indexed by thread or ray
    throughput = state.throughput_and_spatial_index.xyz
    throughput *= albedo * n_dot_l / (VX_PI_F * max(pdf, 1e-6))
    state.throughput_and_spatial_index.xyz = throughput
    state_buffer[thread_id] = state

    # 13. Output Ray
    output_ray_index = InterlockedAdd(output_ray_count[0], 1u, 0) # Assuming atomic counter
    output_ray = path_tracer_ray()
    output_ray.origin_and_path_index = float4(offset_ray(position, normal), asfloat(path_index))
    output_ray.direction = next_ray_dir
    output_ray_buffer[output_ray_index] = output_ray
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** In step 6, `dot(ray_dir, axis_mask)` is used as a divisor. Since `axis_mask` selects one axis, and `ray_dir` is a direction vector, this component should not be zero unless the ray is perfectly parallel to the plane, which is handled by the ray marching logic or results in a miss [S1, unofficial].
*   **PDF Zero:** In step 11, `max(pdf, 1e-6)` is used to prevent division by zero in the throughput calculation [S1, unofficial].
*   **Self-Intersection:** The `offset_ray` function is used to offset the new ray origin to prevent self-intersection [S1, unofficial].

## Subtopics

### [Wavefront Path Tracer with Spatial Hashing](4490-wavefront-path-tracer-with-spatial-hashing.md)
A path tracing method that uses spatial hashing to accumulate radiance estimates before final display. It differs from standard path tracing by decoupling the accumulation buffer from the pixel grid, enabling spatial averaging of radiance estimates across hash buckets.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Not given in the sources.
*   **Strengths:** Uses MIS to combine sun disk and cosine-weighted hemisphere sampling, potentially reducing variance in scenes with strong directional lighting [S1, unofficial]. Decouples accumulation from pixel grid for spatial averaging [S1, unofficial].
*   **Limitations:** Relies on sparse voxel data structure, which may have overhead for very sparse or very dense scenes [S1, unofficial].
*   **When to Use:** Scenes with participating media and strong directional lighting where MIS can improve convergence [S1, unofficial].
*   **When Not to Use:** Scenes without participating media or where simple hemisphere sampling is sufficient [S1, unofficial].

## Background
Origin: The method is part of the `vxray` project by Nelarius [S1, unofficial]. The project uses LLMs to generate C code and bits of shaders [S1, unofficial]. The specific kernel is named `PATH_TRACER_EXTEND_CS` [S1, unofficial].

## Sources
[S1] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
