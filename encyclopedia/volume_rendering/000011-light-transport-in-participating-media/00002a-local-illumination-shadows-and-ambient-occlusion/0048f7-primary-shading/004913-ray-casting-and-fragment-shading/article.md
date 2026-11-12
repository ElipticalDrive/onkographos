# Ray Casting and Fragment Shading

Also called: Fragment Shading with Voxel Lighting

## Overview
This method evaluates shading for traced voxel materials by casting primary rays per fragment to determine surface interaction, then computing direct lighting, ambient occlusion (AO), and indirect lighting contributions. It differs from [Basic Gas Rendering Algorithm](12423-basic-gas-rendering-algorithm.md) by focusing on discrete surface hits and fragment-based shading rather than volumetric integration along the ray, and builds on [Primary Shading](12386-primary-shading.md) by handling the specific ray-casting-to-fragment pipeline for voxel environments.

## How it works
**Inputs:** Fragment position (`in.position.xy`), UV coordinates (`in.uv`), camera inverse matrix (`trace_uniforms.camera_inverse`), voxel texture size (`voxel_uniforms.texture_size`), time (`trace_uniforms.time`), samples count (`trace_uniforms.samples`), and a debug flag (`trace_uniforms.show_ray_steps`).
**Outputs:** Final color (`output_color`), hit normal texture (`normal`), and hit position texture (`position`).

**Data Structures:**
- `hit`: Struct containing `material` (with `rgb`), `pos` (world position), `normal` (world normal), `steps` (u32 ray step count), and `reprojection_pos` (reprojected position).
- `ao`: Vector of 4 floats (`vec4<f32>`) representing interpolated AO values from corners.

**Steps:**
1. **Seed Generation:** Generate a deterministic seed for noise/lighting calculations.
   $$ seed = \text{vec3<u32>}(in.position.xyz) \times 100u + u32(trace\_uniforms.time \times 120.0) \times 15236u $$
2. **Ray Setup:** Convert fragment UV to clip space and unproject to world space.
   $$ clip\_space = \text{vec2}(1.0, -1.0) \times (in.uv \times 2.0 - 1.0) $$
   $$ pos1 = trace\_uniforms.camera\_inverse \times \text{vec4}(clip\_space.x, clip\_space.y, 1.0, 1.0) $$
   $$ dir1 = trace\_uniforms.camera\_inverse \times \text{vec4}(clip\_space.x, clip\_space.y, 0.01, 1.0) $$
   $$ pos = pos1.xyz / pos1.w $$
   $$ dir = \text{normalize}(dir1.xyz / dir1.w - pos) $$
3. **Ray Casting:** Cast the ray into the voxel scene.
   $$ hit = \text{shoot\_ray}(ray, 0.0, 0u) $$
   $$ steps = hit.steps $$
4. **Shading (if hit):**
   - **Direct Lighting:** Calculate direct illumination based on material, position, normal, seed, and samples.
     $$ direct\_lighting = \text{calculate\_direct}(hit.material, hit.pos, hit.normal, seed + 1u, trace\_uniforms.samples) $$
   - **Ambient Occlusion (AO) Coordinates:** Compute texture coordinates for AO lookup.
     $$ texture\_coords = hit.pos \times VOXELS\_PER\_METER + \text{f32}(voxel\_uniforms.texture\_size) / 2.0 $$
   - **AO Interpolation UV:** Calculate UV for bilinear interpolation of AO values.
     $$ uv = \text{glmod}(\text{vec2}(\text{dot}(hit.normal \times texture\_coords.yzx, \text{vec3}(1.0)), \text{dot}(hit.normal \times texture\_coords.zxy, \text{vec3}(1.0))), \text{vec2}(1.0)) $$
   - **Interpolate AO:** Mix the 4 AO values using the calculated UV.
     $$ interpolated\_ao\_pweig = \text{mix}(\text{mix}(ao.z, ao.w, uv.x), \text{mix}(ao.y, ao.x, uv.x), uv.y) $$
   - **AO Gamma Correction:** Apply gamma correction to AO.
     $$ voxel\_ao = \text{pow}(interpolated\_ao\_pweig, 1.0 / 3.0) $$
   - **Indirect Lighting:** Approximate indirect lighting using the gamma-corrected AO.
     $$ indirect\_lighting\_color = \text{vec3}(0.3 \times voxel\_ao) $$
   - **Final Color:** Combine indirect, direct, and material color.
     $$ output\_color = (indirect\_lighting\_color + direct\_lighting.color) \times hit.material.rgb $$
5. **No Hit Case:** If the ray misses, set a default color.
   $$ output\_color = \text{vec3}(0.3) $$
6. **Debug Override:** If enabled, override color with step count visualization.
   $$ \text{if } trace\_uniforms.show\_ray\_steps \neq 0u: output\_color = \text{vec3<f32>}(f32(steps) / 100.0) $$
7. **Clamping:** Ensure no negative color values.
   $$ output\_color = \text{max}(output\_color, \text{vec3}(0.0)) $$
8. **Store Results:** Write hit normal and reprojection position to textures.
   $$ \text{textureStore}(normal, \text{vec2<i32>}(in.position.xy), \text{vec4}(hit.normal, 0.0)) $$
   $$ \text{textureStore}(position, \text{vec2<i32>}(in.position.xy), \text{vec4}(hit.reprojection\_pos, 0.0)) $$
9. **Return:** Output the final color with alpha 1.0.
   $$ \text{return } \text{vec4<f32>}(output\_color, 1.0) $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_casting_fragment_shading(in_position_xy, in_uv, camera_inverse, voxel_texture_size, time, samples, show_ray_steps, voxels_per_meter):
    # 1. Seed Generation
    seed = vec3_u32(in_position_xyz) * 100 + u32(time * 120.0) * 15236

    # 2. Ray Setup
    clip_space = vec2(1.0, -1.0) * (in_uv * 2.0 - 1.0)
    pos1 = camera_inverse * vec4(clip_space.x, clip_space.y, 1.0, 1.0)
    dir1 = camera_inverse * vec4(clip_space.x, clip_space.y, 0.01, 1.0)
    pos = pos1.xyz / pos1.w
    dir = normalize(dir1.xyz / dir1.w - pos)

    # 3. Ray Casting
    hit = shoot_ray(pos, dir, 0.0, 0u)
    steps = hit.steps

    # 4. Shading
    if hit.is_hit:
        # Direct Lighting
        direct_lighting = calculate_direct(hit.material, hit.pos, hit.normal, seed + 1, samples)

        # AO Coordinates
        texture_coords = hit.pos * voxels_per_meter + f32(voxel_texture_size) / 2.0

        # AO Interpolation UV
        uv = glmod(vec2(dot(hit.normal * texture_coords.yzx, vec3(1.0)), dot(hit.normal * texture_coords.zxy, vec3(1.0))), vec2(1.0))

        # Interpolate AO (assuming ao is fetched from texture based on texture_coords)
        ao = fetch_ao_corners(texture_coords) # Not given in sources, assumed helper
        interpolated_ao_pweig = mix(mix(ao.z, ao.w, uv.x), mix(ao.y, ao.x, uv.x), uv.y)

        # AO Gamma
        voxel_ao = pow(interpolated_ao_pweig, 1.0 / 3.0)

        # Indirect Lighting
        indirect_lighting_color = vec3(0.3 * voxel_ao)

        # Combine
        output_color = (indirect_lighting_color + direct_lighting.color) * hit.material.rgb
    else:
        output_color = vec3(0.3)

    # 6. Debug Override
    if show_ray_steps != 0:
        output_color = vec3<f32>(f32(steps) / 100.0)

    # 7. Clamp
    output_color = max(output_color, vec3(0.0))

    # 8. Store Results
    texture_store(normal, vec2<i32>(in_position_xy), vec4(hit.normal, 0.0))
    texture_store(position, vec2<i32>(in_position_xy), vec4(hit.reprojection_pos, 0.0))

    # 9. Return
    return vec4<f32>(output_color, 1.0)
```

**Parameters:**
- `VOXELS_PER_METER`: Not given in the sources.
- `trace_uniforms.samples`: Not given in the sources.
- `trace_uniforms.time`: Not given in the sources.

**Edge Cases and Pitfalls:**
- Division by zero in ray unprojection if `pos1.w` or `dir1.w` is zero. The source code snippet for `slerp` suggests handling degenerate cases (e.g., `theta.sin() == 0.0`), but explicit guards for ray unprojection are not detailed in the primary notes [S1, unofficial].
- Precision issues in `glmod` for UV calculation if coordinates are large.
- The AO interpolation assumes a specific layout of `ao` values (z, w, y, x) which may not be standard bilinear interpolation without clarification on texture sampling [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of ray steps for shading, plus constant time for AO interpolation and lighting calculations.
- **Memory Complexity:** Constant per fragment for temporary variables; requires storage for normal and position textures.
- **Performance:** Depends heavily on the efficiency of `shoot_ray` and `calculate_direct`. The use of fragment shading allows for parallel execution across pixels.
- **Strengths:** Simple implementation, leverages GPU parallelism, provides basic shading and AO.
- **Limitations:** No volumetric effects, limited lighting accuracy (approximate indirect lighting), debug visualization overrides final color.
- **When to use:** Real-time voxel rendering where performance is critical and high-fidelity lighting is not required.
- **When not to use:** Scenes requiring accurate global illumination, subsurface scattering, or volumetric effects.

## Background
This method is part of the `bevy-voxel-engine` project, an unofficial voxel engine for the Bevy game engine [S1, unofficial]. It implements a fragment-based shading approach for voxel materials, integrating ray casting for surface detection and basic lighting models.

## Sources
[S1] ria8651/bevy-voxel-engine, https://github.com/ria8651/bevy-voxel-engine, fetched 2026-10-27 (unofficial)
