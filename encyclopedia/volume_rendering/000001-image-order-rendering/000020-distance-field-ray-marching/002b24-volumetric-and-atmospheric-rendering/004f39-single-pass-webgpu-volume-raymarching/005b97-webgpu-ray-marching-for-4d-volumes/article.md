# WebGPU Ray Marching for 4D Volumes

Also called: Volume4D Viewer Ray Marching, WASM Volume Rendering

## Overview
A real-time volume rendering technique using WebGPU (WGSL) to ray march through 3D spatial data with a temporal dimension (4D), supporting interpolation between time steps and clipping via axis-aligned bounding boxes [S1, unofficial]. It extends the single-pass WebGPU volume raymarching method by adding temporal filtering and explicit 4D data handling, allowing for the visualization of time-varying volumetric fields such as plasma simulations [S1, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
- **Volume Textures:** Two 3D textures (`volume`, `volume_next`) storing scalar field values (e.g., magnetic field magnitude) for consecutive time steps. Format: `texture_3d<f32>` [S1, unofficial].
- **Color Map:** A 2D texture (`cmap`) storing the transfer function (RGB colors) [S1, unofficial].
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
- Binding 0: `volume` (`texture_3d<f32>`)
- Binding 1: `volume_next` (`texture_3d<f32>`)
- Binding 2: `volume_sampler` (`sampler`)
- Binding 3: `camera` uniform buffer (`CameraUniforms`)
- Binding 4: `settings` uniform buffer (`Settings`)

**Bind Group Layout (Group 1):**
- Binding 0: `cmap` (`texture_2d<f32>`)
- Binding 1: `cmap_sampler` (`sampler`)

### Constants
- `FILTER_NEAREST = 0u`
- `FILTER_LINEAR = 1u`
- `PI = 3.1415926535897932384626433832795`
- `TWO_PI = 6.283183095132743846264338327950`
- `early_stopping_t = 1.0 / 255.0`

### Algorithm Steps

1.  **Vertex Shader Setup:**
    Generate two vertices covering the screen using the vertex index.
    -   $xy_x = \text{step}(0.5, \text{f32}(in\_vertex\_index \% 2u == 0u))$
    -   $xy_y = \text{step}(0.5, \text{f32}(in\_vertex\_index < 2u))$
    -   Position: $pos = vec4(xy \times 2.0 - 1.0, 0.0, 1.0)$
    -   TexCoord: $uv = vec2(xy_x, 1.0 - xy_y)$

2.  **Ray Creation:**
    Compute the ray origin and direction from screen coordinates.
    -   Convert screen UV to NDC: $px = uv \times 2.0 - 1.0$
    -   Compute far point in NDC: $far_{ndc} = vec4(px_x, -1.0, 1.0, 1.0)$
    -   Flip Y: $far_{ndc}.y *= -1.0$
    -   Transform to world: $far_w = proj\_inv \times view\_inv \times far_{ndc}$
    -   Normalize W: $far_w.xyz /= far_w.w + 1e-4$
    -   Compute near point in NDC: $near_{ndc} = vec4(px_x, 1.0, 1.0, 1.0)$
    -   Flip Y: $near_{ndc}.y *= -1.0$
    -   Transform to world: $near_w = proj\_inv \times view\_inv \times near_{ndc}$
    -   Normalize W: $near_w.xyz /= near_w.w + 1e-4$
    -   Ray Origin: $ray.orig = near_w.xyz$
    -   Ray Direction: $ray.dir = normalize(far_w.xyz - near_w.xyz)$

3.  **AABB Intersection:**
    Intersect the ray with the volume's axis-aligned bounding box.
    -   $t_{min} = (aabb.min - ray.orig) / ray.dir$
    -   $t_{max} = (aabb.max - ray.orig) / ray.dir$
    -   $t_1 = min(t_{min}, t_{max})$
    -   $t_2 = max(t_{min}, t_{max})$
    -   $t_{near} = max(max(t_1.x, t_1.y), t_1.z)$
    -   $t_{far} = min(min(t_2.x, t_2.y), t_2.z)$
    -   If $t_{near} > t_{far}$, return transparent black `vec3(0.0)` [S1, unofficial].

4.  **Clipping AABB Adjustment:**
    Adjust the intersection bounds based on user-defined clipping planes.
    -   $aabb\_size = aabb.max - aabb.min$
    -   $aabb\_min = aabb.min + (slice\_min \times aabb\_size)$
    -   $aabb\_max = aabb.max - ((1.0 - slice\_max) \times aabb\_size)$
    -   *Note: The source comments indicate 'zxy for tensorf alignment', implying potential coordinate axis permutation in the underlying tensor format, but the math uses standard x/y/z components for the calculation [S1, unofficial].*
    -   Re-calculate $t_{near}$ and $t_{far}$ using the adjusted bounds. If no intersection, return black.

5.  **Ray Initialization:**
    -   $start = max(0.0, t_{near}) + 1e-4$
    -   $ray.orig += start \times ray.dir$
    -   $iters = 0u$
    -   $color = vec3(0.0)$
    -   $transmittance = 1.0$
    -   $distance\_scale = settings.distance\_scale$
    -   $pos = ray.orig$
    -   $step\_size\_g = settings.step\_size$

6.  **Marching Loop:**
    Iterate until early stopping or clipping bounds are exceeded.
    -   **Next Position:**
        -   $sample\_pos\_norm = (pos - aabb.min) / aabb\_size$
        -   $pos += ray.dir \times step\_size\_g$
        -   Return $sample\_pos = vec4(sample\_pos\_norm, step\_size\_g)$
    -   **Volume Sampling:**
        -   Flip Y: $pos\_m = vec3(sample\_pos.x, 1.0 - sample\_pos.y, sample\_pos.z)$
        -   Sample current time step: $sample\_curr = textureSampleLevel(volume, sampler, pos\_m, 0.0).r$
        -   Sample next time step: $sample\_next = textureSampleLevel(volume\_next, sampler, pos\_m, 0.0).r$
    -   **Temporal Filtering:**
        -   If $settings.temporal\_filter == FILTER\_NEAREST$: $value = sample\_curr$
        -   Else ($FILTER\_LINEAR$):
            -   $time\_fraction = fract(settings.time \times f32(settings.time\_steps - 1))$
            -   $value = mix(sample\_curr, sample\_next, time\_fraction)$
    -   **Transfer Function:**
        -   Normalize value: $value\_n = (value - settings.vmin) / (settings.vmax - settings.vmin)$
        -   Sample color map: $sample\_color = textureSampleLevel(cmap, cmap\_sampler, vec2(value\_n, 0.5), 0.0)$
    -   **Absorption Calculation:**
        -   $\sigma = sample\_color.a \times (1.0 - 1e-6)$ *Note: The factor $(1 - 1e-6)$ prevents $\sigma$ from being exactly 1.0 to avoid artifacts [S1, unofficial].*
        -   $a_i = 1.0 - pow(1.0 - \sigma, step\_size \times distance\_scale)$
    -   **Compositing:**
        -   $color += transmittance \times a_i \times sample\_color.rgb$
        -   $transmittance *= 1.0 - a_i$
    -   **Early Stopping:**
        -   If $exp(-transmittance) \le early\_stopping\_t$, break loop.
    -   **Clipping Check:**
        -   If $any(sample\_pos.xyz < settings.clipping.min) \ || \ any(sample\_pos.xyz > settings.clipping.max)$, break loop.
    -   **Increment:**
        -   $iters += 1u$

7.  **Post-Processing:**
    -   If $settings.gamma\_correction == 1u$, apply gamma correction to $color$.
    -   **Gamma Correction (Linear to sRGB):**
        -   $cutoff = color.rgb < vec3(0.0031308)$
        -   $higher = vec3(1.055) \times pow(color.rgb, vec3(1.0 / 2.4)) - 0.055$
        -   $lower = color.rgb \times 12.92$
        -   $color.rgb = mix(higher, lower, cutoff)$

8.  **Render Pass:**
    -   Bind Group 0: Volume textures, samplers, camera uniforms, settings.
    -   Bind Group 1: Color map texture, sampler.
    -   Draw 4 vertices (0..4) for 1 instance.

### Pseudocode (WGSL-like)

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```wgsl
// Constants
const FILTER_NEAREST: u32 = 0u;
const FILTER_LINEAR: u32 = 1u;
const EARLY_STOPPING_T: f32 = 1.0 / 255.0;

struct Aabb {
    min: vec3<f32>,
    max: vec3<f32>,
};

struct CameraUniforms {
    view: mat4x4<f32>,
    view_inv: mat4x4<f32>,
    proj: mat4x4<f32>,
    proj_inv: mat4x4<f32>,
};

struct Settings {
    volume_aabb: Aabb,
    clipping: Aabb,
    time: f32,
    time_steps: u32,
    step_size: f32,
    temporal_filter: u32,
    distance_scale: f32,
    vmin: f32,
    vmax: f32,
    gamma_correction: u32,
};

@group(0) @binding(0) var volume: texture_3d<f32>;
@group(0) @binding(1) var volume_next: texture_3d<f32>;
@group(0) @binding(2) var volume_sampler: sampler;
@group(0) @binding(3) var camera: CameraUniforms;
@group(0) @binding(4) var settings: Settings;

@group(1) @binding(0) var cmap: texture_2d<f32>;
@group(1) @binding(1) var cmap_sampler: sampler;

fn linear_to_srgb(color: vec3<f32>) -> vec3<f32> {
    let cutoff = color.rgb < vec3<f32>(0.0031308);
    let higher = vec3<f32>(1.055) * pow(color.rgb, vec3<f32>(1.0 / 2.4)) - vec3<f32>(0.055);
    let lower = color.rgb * vec3<f32>(12.92);
    return mix(higher, lower, cutoff);
}

fn sample_volume(pos_norm: vec3<f32>, time_frac: f32) -> vec3<f32> {
    let pos_m = vec3<f32>(pos_norm.x, 1.0 - pos_norm.y, pos_norm.z);
    let sample_curr = textureSampleLevel(volume, volume_sampler, pos_m, 0.0).r;
    let sample_next = textureSampleLevel(volume_next, volume_sampler, pos_m, 0.0).r;
    
    var value: f32;
    if (settings.temporal_filter == FILTER_NEAREST) {
        value = sample_curr;
    } else {
        value = mix(sample_curr, sample_next, time_frac);
    }
    
    let value_n = (value - settings.vmin) / (settings.vmax - settings.vmin);
    return textureSampleLevel(cmap, cmap_sampler, vec2<f32>(value_n, 0.5), 0.0).rgb;
}

fn get_alpha(value: f32) -> f32 {
    let value_n = (value - settings.vmin) / (settings.vmax - settings.vmin);
    let sample_color = textureSampleLevel(cmap, cmap_sampler, vec2<f32>(value_n, 0.5), 0.0);
    return sample_color.a * (1.0 - 1e-6);
}

@vertex
fn vs_main(@builtin(vertex_index) in_vertex_index: u32) -> @builtin(position) vec4<f32> {
    let xy_x = f32(in_vertex_index % 2u == 0u);
    let xy_y = f32(in_vertex_index < 2u);
    let xy = vec2<f32>(xy_x, xy_y);
    return vec4<f32>(xy * 2.0 - 1.0, 0.0, 1.0);
}

@fragment
fn fs_main(@builtin(position) frag_coord: vec4<f32>) -> @location(0) vec3<f32> {
    let uv = vec2<f32>(frag_coord.x / 1000.0, 1.0 - frag_coord.y / 1000.0); // Assuming resolution passed or derived
    // Note: In the source, UV is derived from vertex index in VS, passed as TexCoord.
    // Here we assume standard full screen quad setup where uv is passed from VS.
    // Reconstructing from source logic:
    let px = uv * 2.0 - 1.0;
    
    // Ray Creation
    var far_ndc = vec4<f32>(px.x, -1.0, 1.0, 1.0);
    far_ndc.y *= -1.0;
    var far_w = camera.proj_inv * camera.view_inv * far_ndc;
    far_w.xyz /= far_w.w + 1e-4;
    
    var near_ndc = vec4<f32>(px.x, 1.0, 1.0, 1.0);
    near_ndc.y *= -1.0;
    var near_w = camera.proj_inv * camera.view_inv * near_ndc;
    near_w.xyz /= near_w.w + 1e-4;
    
    var ray_orig = near_w.xyz;
    var ray_dir = normalize(far_w.xyz - near_w.xyz);
    
    // AABB Intersection
    let t_min = (settings.volume_aabb.min - ray_orig) / ray_dir;
    let t_max = (settings.volume_aabb.max - ray_orig) / ray_dir;
    let t1 = min(t_min, t_max);
    let t2 = max(t_min, t_max);
    var t_near = max(max(t1.x, t1.y), t1.z);
    var t_far = min(min(t2.x, t2.y), t2.z);
    
    // Clipping Adjustment
    let aabb_size = settings.volume_aabb.max - settings.volume_aabb.min;
    // Note: slice_min and slice_max are not explicitly defined in the provided notes as inputs,
    // but are used in the clipping calculation. Assuming they are part of 'settings' or derived.
    // Based on source: aabb_min = (aabb.min + (slice_min * aabb_size))
    // We assume settings.clipping.min/max are pre-calculated or slice_min/max are 0/1 if not clipped.
    // For implementation, we use the provided clipping bounds directly if they represent the active volume.
    // The source calculates adjusted min/max from aabb.min/max and slice parameters.
    // Let's assume settings.clipping holds the final active bounds.
    
    // Re-intersect with clipping bounds if they differ from volume_aabb
    // For simplicity, if clipping is active, we use settings.clipping for t_near/t_far calculation
    // However, the source logic suggests adjusting t_near/t_far based on slice_min/max.
    // Let's stick to the source's explicit formula:
    let slice_min = 0.0; // Placeholder if not provided in settings, typically 0
    let slice_max = 1.0; // Placeholder if not provided in settings, typically 1
    let adj_min = settings.volume_aabb.min + (slice_min * aabb_size);
    let adj_max = settings.volume_aabb.max - ((1.0 - slice_max) * aabb_size);
    
    let t_min_clip = (adj_min - ray_orig) / ray_dir;
    let t_max_clip = (adj_max - ray_orig) / ray_dir;
    let t1_clip = min(t_min_clip, t_max_clip);
    let t2_clip = max(t_min_clip, t_max_clip);
    t_near = max(max(t1_clip.x, t1_clip.y), t1_clip.z);
    t_far = min(min(t2_clip.x, t2_clip.y), t2_clip.z);
    
    if (t_near > t_far) {
        return vec3<f32>(0.0);
    }
    
    // Initialization
    var start = max(0.0, t_near) + 1e-4;
    ray_orig += start * ray_dir;
    var pos = ray_orig;
    var color = vec3<f32>(0.0);
    var transmittance = 1.0;
    let step_size_g = settings.step_size;
    let distance_scale = settings.distance_scale;
    
    // Time Fraction
    let time_frac = fract(settings.time * f32(settings.time_steps - 1u));
    
    // Marching Loop
    for (var i: u32 = 0u; i < 1000u; i = i + 1u) { // Limit iterations to prevent infinite loops
        let sample_pos_norm = (pos - settings.volume_aabb.min) / aabb_size;
        
        // Check clipping bounds for current sample
        if (any(sample_pos_norm < settings.clipping.min) || any(sample_pos_norm > settings.clipping.max)) {
            break;
        }
        
        let sample_color = sample_volume(sample_pos_norm, time_frac);
        let sigma = sample_color.a * (1.0 - 1e-6);
        let a_i = 1.0 - pow(1.0 - sigma, step_size_g * distance_scale);
        
        color += transmittance * a_i * sample_color.rgb;
        transmittance *= 1.0 - a_i;
        
        if (exp(-transmittance) <= EARLY_STOPPING_T) {
            break;
        }
        
        pos += ray_dir * step_size_g;
    }
    
    if (settings.gamma_correction == 1u) {
        color = linear_to_srgb(color);
    }
    
    return color;
}
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of marching steps per pixel. The number of steps is determined by the volume depth divided by `step_size`.
- **Memory Complexity:** Constant per pixel, plus storage for the two 3D volume textures and the 2D color map texture.
- **Performance:** Real-time performance is achieved on WebGPU-compatible hardware. Performance scales with resolution and the number of time steps (though only two textures are loaded at once).
- **Strengths:** Supports 4D data (3D space + time) with smooth interpolation. Efficient early-alpha termination. Simple AABB clipping.
- **Limitations:** Fixed step size may lead to aliasing or slow convergence in sparse regions. No adaptive stepping. Clipping is axis-aligned.
- **When to use:** Real-time visualization of time-varying volumetric data in a browser.
- **When not to use:** High-fidelity offline rendering requiring adaptive sampling or complex transfer functions not supported by 1D lookup.

## Background
This method is part of the `volume4d-viewer` project by Jonathon Brunson, designed for interactive visualization of space and time volumetric data using WebGPU and Rust [S1, unofficial]. It builds upon the single-pass WebGPU volume raymarching technique by adding temporal filtering capabilities [S1, unofficial].

## Sources
[S1] jonathonb21/volume4d-viewer, https://github.com/jonathonb21/volume4d-viewer, fetched 2026-10-26 (unofficial)
