# Path Tracing Extension with MIS

Also called: voxel path tracing

## Overview
This method extends a path tracing kernel by marching rays through sparse voxel data, shading intersections using albedo and geometric normals, and generating new ray directions via Multiple Importance Sampling (MIS) between sun disk and cosine-weighted hemisphere sampling [S1, unofficial]. It differs from standard path tracing by decoupling the accumulation buffer from the pixel grid (via spatial hashing in its wavefront variant) and explicitly handling sparse voxel traversal for volume shading [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input Ray Buffer:** `StructuredBuffer<path_tracer_ray>`. Each ray contains `origin_and_path_index` (float4, where xyz is the origin and w is the path index as a uint) and `direction` (float3) [S1, unofficial].
*   **Path State Buffer:** `RWStructuredBuffer<path_tracer_path_state>`. Stores `radiance` (float3) and `throughput_and_spatial_index` (float3) [S1, unofficial].
*   **Voxel Data:** `Texture3D<uint> voxels`. Each texel is a uint index into the palette buffer [S1, unofficial].
*   **Palette:** `StructuredBuffer<uint> palette_rgba`. Each uint is a packed RGBA color. Albedo is extracted via `unpack_albedo(palette_rgba[voxels.Load(...)].r).rgb` [S1, unofficial].
*   **Output Ray Buffer:** `RWStructuredBuffer<path_tracer_ray>`. Stores extended rays for the next pass [S1, unofficial].
*   **Uniforms:** `sun_direction` (float3), `transmitted_sun_color` (float3), `cos_theta_max` (float, derived from solar radius), `bounce` (uint), `sample_ordinals` (array of uints) [S1, unofficial].

**Data Structures:**
*   `path_tracer_ray`:
    *   `origin_and_path_index`: float4
    *   `direction`: float3
*   `path_tracer_path_state`:
    *   `radiance`: float3
    *   `throughput_and_spatial_index`: float3

**Steps:**
1.  **Dispatch Check:** If `thread_id >= input_ray_count[0]`, return [S1, unofficial].
2.  **Load Ray:** Retrieve `ray = input_ray_buffer[thread_id]`. Extract `ray_origin = ray.origin_and_path_index.xyz`, `ray_dir = ray.direction`, `path_index = asuint(ray.origin_and_path_index.w)` [S1, unofficial].
3.  **March Ray:** Perform sparse voxel ray marching: `packed_cell = sparse_ray_march(ray_origin, ray_dir)` [S1, unofficial].
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

**Parameters:**
*   `VX_PI_F`: The value of Pi used in PDF calculation. Exact value not given, but used as divisor [S1, unofficial].
*   `VX_SKY_SOLAR_RADIUS_RAD`: Solar radius in radians. Defined as `1.5f * 0.0044506f` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Zero PDF:** The PDF calculation includes a `max(pdf, 1e-6)` guard to prevent division by zero [S1, unofficial].
*   **Miss Handling:** If the ray misses all voxels, sky radiance is added directly to the path state [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def path_tracer_extend(thread_id, input_ray_count, input_ray_buffer, output_ray_buffer, output_ray_count, state_buffer, voxels, palette_rgba, uniforms, sample_ordinals):
    if thread_id >= input_ray_count[0]:
        return

    ray = input_ray_buffer[thread_id]
    ray_origin = ray.origin_and_path_index[:3]
    ray_dir = ray.direction
    path_index = int(ray.origin_and_path_index[3])

    packed_cell = sparse_ray_march(ray_origin, ray_dir)

    if packed_cell is None:
        state = state_buffer[thread_id]
        state.radiance += state.throughput * sky_radiance(ray_dir, uniforms.transmitted_sun_color)
        state_buffer[thread_id] = state
        return

    cell = unpack_voxel_cell(packed_cell)
    axis_index = packed_cell >> 30
    axis_mask = [1.0 if axis_index == i else 0.0 for i in range(3)]
    normal = [-math.copysign(1.0, ray_dir[i]) * axis_mask[i] for i in range(3)]

    entry_plane = [cell[i] + (1.0 if ray_dir[i] < 0.0 else 0.0) for i in range(3)]
    distance = sum((entry_plane[i] - ray_origin[i]) * axis_mask[i] for i in range(3)) / sum(ray_dir[i] * axis_mask[i] for i in range(3))
    position = [ray_origin[i] + distance * ray_dir[i] for i in range(3)]

    albedo = unpack_albedo(palette_rgba[voxels[(cell[0], cell[1], cell[2], 0)]])

    sample_index = sample_ordinals[path_index] - 1
    samples = halton_sample_3d(sample_index, uniforms.bounce, path_index)
    u = samples[:2]
    sample_sun = samples[2] < 0.5

    if sample_sun:
        local_dir = sample_cone(u, uniforms.cos_theta_max)
        next_ray_dir = orient_sample_direction(local_dir, uniforms.sun_direction)
    else:
        local_dir = sample_cosine_weighted_hemisphere(u)
        next_ray_dir = orient_axis_aligned_sample_direction(local_dir, normal)

    n_dot_l = max(sum(normal[i] * next_ray_dir[i] for i in range(3)), 0.0)
    if n_dot_l == 0.0:
        return

    pdf = 0.5 * (pdf_cone(sum(uniforms.sun_direction[i] * next_ray_dir[i] for i in range(3)), uniforms.cos_theta_max) + pdf_cosine_weighted_hemisphere(n_dot_l))

    state = state_buffer[thread_id]
    throughput = state.throughput
    throughput *= albedo * n_dot_l / (math.pi * max(pdf, 1e-6))
    state.throughput = throughput
    state_buffer[thread_id] = state

    output_ray_index = output_ray_count[0]
    output_ray_count[0] += 1
    output_ray = PathTracerRay()
    output_ray.origin_and_path_index = offset_ray(position, normal) + [float(path_index)]
    output_ray.direction = next_ray_dir
    output_ray_buffer[output_ray_index] = output_ray
```

## Subtopics

### [Wavefront Path Tracer with Spatial Hashing](4490-wavefront-path-tracer-with-spatial-hashing.md)
A path tracing method that uses spatial hashing to accumulate radiance, allowing for denoising via spatial filtering. It differs from standard path tracing by decoupling the accumulation buffer from the pixel grid, enabling spatial averaging of radiance estimates before final display.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Efficiently handles sparse voxel data and uses MIS between sun disk and cosine-weighted hemisphere sampling to reduce variance in lighting estimates [S1, unofficial].
*   **Limitations:** Requires a sparse voxel data structure and a palette-based albedo lookup [S1, unofficial].
*   **When to use:** Rendering scenes with sparse voxel data where efficient lighting estimation is needed [S1, unofficial].
*   **When not to use:** Scenes with dense voxel data or when MIS overhead is not justified [S1, unofficial].

## Background
Originating from the `vxray` project by Nelarius, this method is part of a broader effort to accelerate voxel rendering using path tracing techniques [S1, unofficial]. It builds on principles of ray tracing and volume rendering, specifically focusing on efficient traversal and shading of sparse voxel grids [S1, unofficial].

## Sources
[S1] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
