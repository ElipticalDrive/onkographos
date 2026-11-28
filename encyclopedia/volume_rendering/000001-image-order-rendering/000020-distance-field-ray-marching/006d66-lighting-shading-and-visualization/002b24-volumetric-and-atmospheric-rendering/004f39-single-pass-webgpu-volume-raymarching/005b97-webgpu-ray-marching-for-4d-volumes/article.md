# WebGPU Ray Marching for 4D Volumes

Also called: Volume4D Viewer Ray Marching, WASM Volume Rendering

## Overview
A real-time volume rendering technique using WebGPU (WGSL) to ray march through 3D spatial data with a temporal dimension (4D), supporting interpolation between time steps and clipping via axis-aligned bounding boxes [S1, unofficial]. It extends the single-pass WebGPU volume raymarching method by adding temporal filtering and explicit 4D data handling, allowing for the visualization of time-varying volumetric fields such as plasma simulations [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- **Volume Textures:** Two 3D textures (`volume`, `volume_next`) storing scalar field values for consecutive time steps, formatted as `texture_3d<f32>` [S1, unofficial].
- **Color Map:** A 2D texture (`cmap`) storing the transfer function (RGB colors), formatted as `texture_2d<f32>` [S1, unofficial].
- **Camera Uniforms:** View, view inverse, projection, and projection inverse matrices [S1, unofficial].
- **Settings Uniform:** AABB bounds, clipping bounds, time parameters, step size, temporal filter mode, distance scale, value range (`vmin`, `vmax`), and gamma correction flag [S1, unofficial].
- **Screen Coordinates:** Vertex index used to generate a full-screen quad [S1, unofficial].

**Output:**
- **Color Buffer:** A 3-component float color (`vec3<f32>`) for each pixel, optionally converted to sRGB [S1, unofficial].

### Data Structures
The following structures are passed via uniform buffers or bind groups in WGSL:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```wgsl
struct Aabb {
    min: vec3<f32>, // Aligned to 16 bytes
    max: vec3<f32>, // Aligned to 16 bytes
}

struct CameraUniforms {
    view: mat4x4<f32>,
    view_inv: mat4x4<f32>,
    proj: mat4x4<f32>,
    proj_inv: mat4x4<f32>,
}

struct Settings {
    volume_aabb: Aabb,
    clipping: Aabb,
    time: f32,
    time_steps: u32,
    step_size: f32,
    temporal_filter: u32, // 0: NEAREST, 1: LINEAR
    distance_scale: f32,
    vmin: f32,
    vmax: f32,
    gamma_correction: u32,
}

struct Ray {
    orig: vec3<f32>,
    dir: vec3<f32>,
}
```

**Bind Group Layout (Group 0):**
- Binding 0: `volume` (`texture_3d<f32>`) [S1, unofficial]
- Binding 1: `volume_next` (`texture_3d<f32>`) [S1, unofficial]
- Binding 2: `volume_sampler` (`sampler`) [S1, unofficial]
- Binding 3: `camera` uniform buffer (`CameraUniforms`) [S1, unofficial]
- Binding 4: `settings` uniform buffer (`Settings`) [S1, unofficial]

**Bind Group Layout (Group 1):**
- Binding 0: `cmap` (`texture_2d<f32>`) [S1, unofficial]
- Binding 1: `cmap_sampler` (`sampler`) [S1, unofficial]

### Constants
- `FILTER_NEAREST = 0u` [S1, unofficial]
- `FILTER_LINEAR = 1u` [S1, unofficial]
- `PI = 3.1415926535897932384626433832795` [S1, unofficial]
- `TWO_PI = 6.283185307179586476925286766559` [S1, unofficial]
- `early_stopping_t = 1. / 255.` [S1, unofficial]

### Algorithm Steps

1. **Vertex Shader Setup:** Generate two vertices covering the screen using vertex index. `xy = vec2<f32>(f32(in_vertex_index % 2u == 0u), f32(in_vertex_index < 2u))`. Position: `vec4<f32>(xy * 2. - 1., 0., 1.)`. TexCoord: `vec2<f32>(xy.x, 1. - xy.y)` [S1, unofficial].
2. **Ray Creation:** Compute far point in NDC: `far = vec4<f32>((px * 2. - 1.), -1., 1.)`. Flip Y: `far.y *= -1.`. Transform to world: `far_w = view_inv * proj_inv * far`. Normalize W: `far_w /= far_w.w + 1e-4` to avoid division by zero [S1, unofficial]. Compute near point: `near = vec4<f32>((px * 2. - 1.), 1., 1.)`. Flip Y: `near.y *= -1.`. Transform to world: `near_w = view_inv * proj_inv * near`. Normalize W: `near_w /= near_w.w + 1e-4` [S1, unofficial]. Ray origin: `near_w.xyz`. Direction: `normalize(far_w.xyz - near_w.xyz)` [S1, unofficial].
3. **AABB Intersection:** `tMin = (box_min - ray.orig) / ray.dir`. `tMax = (box_max - ray.orig) / ray.dir`. `t1 = min(tMin, tMax)`. `t2 = max(tMin, tMax)`. `tNear = max(max(t1.x, t1.y), t1.z)`. `tFar = min(min(t2.x, t2.y), t2.z)` [S1, unofficial]. If `tNear > tFar`, return transparent black `vec4<f32>(0.)` [S1, unofficial].
4. **Clipping AABB Adjustment:** `aabb_size = aabb.max - aabb.min`. `aabb_min = aabb.min + (slice_min * aabb_size)`. `aabb_max = aabb.max - ((1. - slice_max) * aabb_size)` [S1, unofficial]. Note: Source comments indicate 'zxy for tensorf alignment', implying potential coordinate axis permutation in the underlying tensor format [S1, unofficial]. Re-calculate `tNear` and `tFar` with adjusted bounds. If no intersection, return black [S1, unofficial].
5. **Ray Initialization:** `start = max(0., tNear) + 1e-4`. `ray.orig += start * ray.dir`. `iters = 0u`. `color = vec3<f32>(0.)`. `transmittance = 1.`. `distance_scale = settings.distance_scale`. `pos = ray.orig`. `early_stopping_t = 1. / 255.`. `step_size_g = settings.step_size` [S1, unofficial].
6. **Marching Loop:** Call `next_pos(&pos, step_size_g, ray.dir)` to get `sample_pos` (vec4 containing normalized sample position and step_size). Extract `step_size = sample_pos.w`. Sample volume at `sample_pos.xyz`. Apply transfer function. `sigma = color_tf.a * (1. - 1e-6)` to prevent artifacts from sigma exactly one [S1, unofficial]. `a_i = 1. - pow(1. - sigma, step_size * distance_scale)`. `color += transmittance * a_i * sample_color`. `transmittance *= 1. - a_i`. Early stopping: `if exp(-transmittance) <= early_stopping_t, break` [S1, unofficial]. Clipping check: `slice_test = any(sample_pos.xyz < settings.clipping.min) || any(sample_pos.xyz > settings.clipping.max)`. If true, break [S1, unofficial]. Increment `iters` by `1u` [S1, unofficial].
7. **Next Position Calculation:** `sample_pos = ((*pos) - aabb.min) / aabb_size`. `*pos += ray_dir * step_size`. Return `vec4<f32>(sample_pos, step_size)` [S1, unofficial].
8. **Temporal Sampling:** If `temporal_filter == FILTER_NEAREST`, return `sample_curr`. Else, `time_fraction = fract(settings.time * f32(settings.time_steps - 1))`. Return `mix(sample_curr, sample_next, time_fraction)` [S1, unofficial].
9. **Volume Sampling:** Flip Y: `pos_m = vec3<f32>(pos.x, 1. - pos.y)`. `sample_curr = textureSampleLevel(volume, volume_sampler, pos_m, 0.).r`. `sample_next = textureSampleLevel(volume_next, volume_sampler, pos_m, 0.).r` [S1, unofficial].
10. **Color Map Sampling:** `value_n = (value - settings.vmin) / (settings.vmax - settings.vmin)`. `textureSampleLevel(cmap, cmap_sampler, vec2<f32>(value_n, 0.5), 0.)` [S1, unofficial].
11. **Gamma Correction:** If `settings.gamma_correction == 1u`, apply linear to sRGB conversion [S1, unofficial]. Cutoff: `color.rgb < vec3<f32>(0.0031308)`. Higher: `vec3<f32>(1.055) * pow(color.rgb, vec3<f32>(1.0 / 2.4)) - 0.055`. Lower: `color.rgb * 12.92`. Return `mix(higher, lower, cutoff)` with original alpha [S1, unofficial].
12. **Render Pass:** Set bind group 0 and bind group 1. Draw 4 vertices (0..4) for 1 instance [S1, unofficial].

### Pseudocode (Python-like)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
FILTER_NEAREST = 0
FILTER_LINEAR = 1
EARLY_STOPPING_T = 1.0 / 255.0

def linear_to_srgb(color_rgb):
    cutoff = color_rgb < 0.0031308
    higher = 1.055 * pow(color_rgb, 1.0 / 2.4) - 0.055
    lower = color_rgb * 12.92
    return mix(higher, lower, cutoff)

def sample_volume(pos_norm, time_frac, settings):
    pos_m = vec3(pos_norm.x, 1.0 - pos_norm.y, pos_norm.z)
    sample_curr = texture_sample_level(volume, volume_sampler, pos_m, 0.0).r
    sample_next = texture_sample_level(volume_next, volume_sampler, pos_m, 0.0).r
    if settings.temporal_filter == FILTER_NEAREST:
        value = sample_curr
    else:
        time_fraction = fract(settings.time * float(settings.time_steps - 1))
        value = mix(sample_curr, sample_next, time_fraction)
    value_n = (value - settings.vmin) / (settings.vmax - settings.vmin)
    return texture_sample_level(cmap, cmap_sampler, vec2(value_n, 0.5), 0.0)

def compute_ray(px, camera):
    far_ndc = vec4(px * 2.0 - 1.0, -1.0, 1.0)
    far_ndc.y *= -1.0
    far_w = camera.view_inv * camera.proj_inv * far_ndc
    far_w.xyz /= far_w.w + 1e-4
    near_ndc = vec4(px * 2.0 - 1.0, 1.0, 1.0)
    near_ndc.y *= -1.0
    near_w = camera.view_inv * camera.proj_inv * near_ndc
    near_w.xyz /= near_w.w + 1e-4
    ray_orig = near_w.xyz
    ray_dir = normalize(far_w.xyz - near_w.xyz)
    return ray_orig, ray_dir

def intersect_aabb(ray_orig, ray_dir, aabb_min, aabb_max):
    t_min = (aabb_min - ray_orig) / ray_dir
    t_max = (aabb_max - ray_orig) / ray_dir
    t1 = min(t_min, t_max)
    t2 = max(t_min, t_max)
    t_near = max(max(t1.x, t1.y), t1.z)
    t_far = min(min(t2.x, t2.y), t2.z)
    return t_near, t_far

def fragment_shader(frag_coord, camera, settings):
    # Vertex setup generates uv from vertex index
    uv = vec2(frag_coord.x, 1.0 - frag_coord.y) # Simplified from source quad generation
    px = uv * 2.0 - 1.0
    
    ray_orig, ray_dir = compute_ray(px, camera)
    
    # Initial AABB intersection
    t_near, t_far = intersect_aabb(ray_orig, ray_dir, settings.volume_aabb.min, settings.volume_aabb.max)
    if t_near > t_far:
        return vec4(0.0) # Transparent black [S1, unofficial]
    
    # Clipping adjustment
    aabb_size = settings.volume_aabb.max - settings.volume_aabb.min
    # slice_min and slice_max are clipping parameters; exact source not specified in settings struct [S1, unofficial]
    adj_min = settings.volume_aabb.min + (slice_min * aabb_size)
    adj_max = settings.volume_aabb.max - ((1.0 - slice_max) * aabb_size)
    t_near, t_far = intersect_aabb(ray_orig, ray_dir, adj_min, adj_max)
    if t_near > t_far:
        return vec3(0.0)
    
    start = max(0.0, t_near) + 1e-4
    ray_orig += start * ray_dir
    pos = ray_orig
    color = vec3(0.0)
    transmittance = 1.0
    step_size_g = settings.step_size
    distance_scale = settings.distance_scale
    
    time_frac = fract(settings.time * float(settings.time_steps - 1))
    
    for _ in range(10000): # Iteration limit implementation-dependent
        sample_pos_norm = (pos - settings.volume_aabb.min) / aabb_size
        
        # Clipping check
        if any(sample_pos_norm < settings.clipping.min) or any(sample_pos_norm > settings.clipping.max):
            break
            
        sample_color = sample_volume(sample_pos_norm, time_frac, settings)
        sigma = sample_color.a * (1.0 - 1e-6)
        a_i = 1.0 - pow(1.0 - sigma, step_size_g * distance_scale)
        
        color += transmittance * a_i * sample_color.rgb
        transmittance *= 1.0 - a_i
        
        if exp(-transmittance) <= EARLY_STOPPING_T:
            break
            
        pos += ray_dir * step_size_g
        
    if settings.gamma_correction == 1:
        color = linear_to_srgb(color)
        
    return color
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of marching steps per pixel. The number of steps is determined by the volume depth divided by `step_size` [S1, unofficial].
- **Memory Complexity:** Constant per pixel, plus storage for the two 3D volume textures and the 2D color map texture [S1, unofficial].
- **Performance:** Real-time performance is achieved on WebGPU-compatible hardware. Performance scales with resolution and the number of time steps (though only two textures are loaded at once) [S1, unofficial].
- **Strengths:** Supports 4D data (3D space + time) with smooth interpolation. Efficient early-alpha termination. Simple AABB clipping [S1, unofficial].
- **Limitations:** Fixed step size may lead to aliasing or slow convergence in sparse regions. No adaptive stepping. Clipping is axis-aligned [S1, unofficial].
- **When to use:** Real-time visualization of time-varying volumetric data in a browser [S1, unofficial].
- **When not to use:** High-fidelity offline rendering requiring adaptive sampling or complex transfer functions not supported by 1D lookup [S1, unofficial].

## Background
This method is part of the `volume4d-viewer` project by Jonathon Brunson, designed for interactive visualization of space and time volumetric data using WebGPU and Rust [S1, unofficial]. It builds upon the single-pass WebGPU volume raymarching technique by adding temporal filtering capabilities [S1, unofficial].

## Sources
[S1] jonathonb21/volume4d-viewer, https://github.com/jonathonb21/volume4d-viewer, fetched 2026-10-26 (unofficial)
