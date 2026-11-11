# Procedural Volumetric Cloud Rendering

Also called: Nubis³, Nubis Cubed, Nubis Evolved, Volumetric Cloud Renderer, Volumetric Cloud Rendering, Real-time Volumetric Cloud Rendering

## Overview
Procedural Volumetric Cloud Rendering is a real-time image-order ray marching technique that renders volumetric clouds as a spherical shell or bounded volume using procedural noise for density shaping and multiple-scattering approximations for lighting. It solves the problem of generating physically plausible, dynamically moving cloudscapes with accurate shadowing and atmospheric perspective without relying on pre-baked textures or heavy voxel grids. It is distinguished from sibling methods by its specific combination of Nubis-style dimensional noise profiles for shape, Frostbite-style energy-conserving integration for lighting, and often a spherical shell geometry that conforms to planetary curvature, allowing the camera to be inside, below, or above the cloud layer [S1, S3, unofficial].

## How it works
The method operates within the image-order ray marching framework defined by [Ray Marching and Discretization](2788-ray-marching-and-discretization.md).

### Inputs and Outputs
**Inputs:**
*   Camera position ($C_{pos}$), view direction, and projection matrix.
*   Sun direction ($S_{dir}$) and color ($S_{color}$).
*   Procedural noise textures:
    *   **Shape Texture:** A 3D texture (e.g., $128^3$) containing Perlin-Worley noise for the base cloud structure [S3, unofficial].
    *   **Detail Texture:** A 3D texture (e.g., $32^3$) containing Worley noise for fine erosion [S3, unofficial].
    *   **Weather Map:** A 2D texture (e.g., $512^2$) providing coverage and cloud type (stratus, cumulus, etc.) per region [S3, unofficial].
*   Layer parameters: Base height ($H_{base}$), thickness ($H_{thick}$), wind vector ($W_{vec}$), and time ($t$) [S3, unofficial].
*   Solid geometry depth/color buffers (for compositing with terrain) [S2, unofficial].

**Outputs:**
*   Fragment color: In-scattered radiance ($L_{in}$) and transmittance ($T$) [S3, unofficial].
*   Depth: Transmittance-weighted mean depth for temporal reprojection [S3, unofficial].

### Data Structures
*   **Ray:** Defined by origin $O$ and direction $D$.
*   **Cloud Layer:** Defined as a spherical shell with inner radius $R_{inner}$ and outer radius $R_{outer}$, or an Axis-Aligned Bounding Box (AABB) [S2, S3, unofficial].
*   **Noise Sampling:** Positions are transformed using wind offset and scale factors. The shape noise is typically sampled at a distance-adaptive mip level [S1, S3, unofficial].

### Steps

1.  **Ray Setup and Intersection:**
    *   Construct the ray from the camera position and screen UV coordinates.
    *   Intersect the ray with the cloud volume.
        *   *Spherical Shell:* Calculate intersection with outer sphere ($t_{outer\_near}, t_{outer\_far}$) and inner sphere ($t_{inner\_near}, t_{inner\_far}$). The integration interval is $[t_{near}, t_{far}]$ where $t_{near} = \max(t_{outer\_near}, t_{inner\_near})$ and $t_{far} = \min(t_{outer\_far}, t_{inner\_far})$ [S3, unofficial].
        *   *AABB:* Use slab method intersection. Clamp $t_{near}$ to $\max(0, t_{hit.x})$ and $t_{far}$ to $\min(t_{hit.y}, t_{solid})$ if compositing with terrain [S2, unofficial].

2.  **Discretization:**
    *   Calculate step size $\Delta t$. This can be fixed or adaptive based on density.
    *   Apply sub-pixel jitter using a Halton sequence or interleaved gradient noise to facilitate temporal reprojection [S3, unofficial].

3.  **Integration Loop:**
    For each step $i$ at position $P_i = O + t_i D$:
    *   **Density Sampling:**
        *   Compute wind-offset position: $P_{wind} = P_i + W_{vec} \cdot t$.
        *   Sample Shape Texture $N_{shape}$ and Detail Texture $N_{detail}$.
        *   Calculate base density from shape noise.
        *   Apply erosion using detail noise: $D_{final} = \max(D_{base} - \text{erosion}(N_{detail}) \cdot \text{coverage}, 0)$ [S3, unofficial].
        *   Apply vertical falloff based on height relative to $H_{base}$ and $H_{thick}$ [S2, S3, unofficial].
    *   **Optical Depth Calculation:**
        *   Sample the density along the ray direction towards the sun (typically 6 samples) to compute optical depth $\tau_{sun}$ [S3, unofficial].
    *   **Lighting (In-Scattering):**
        *   Compute phase function value $P(\theta)$ using a two-lobe Henyey-Greenstein or Mie phase function, where $\theta$ is the angle between ray direction and sun direction [S1, S3, unofficial].
        *   Calculate multiple-scattering approximation. This involves evaluating attenuation, contribution, and phase eccentricity for multiple octaves [S3, unofficial].
        *   Compute ambient term: Interpolate between top ambient color (blue) and bottom ambient color (grey) based on height [S3, unofficial].
        *   Compute powder term: Darken back-lit edges [S3, unofficial].
        *   Integrate in-scattered radiance $L_{in}$ analytically over the step size to ensure step-size independence [S3, unofficial].
    *   **Transmittance Update:**
        *   Update transmittance: $T \leftarrow T \cdot e^{-D_{final} \cdot \Delta t}$ [S1, S2, unofficial].
    *   **Color Accumulation:**
        *   Accumulate color: $C \leftarrow C + L_{in} \cdot T \cdot \Delta t$ [S1, S2, unofficial].

4.  **Termination:**
    *   Stop if $T < 0.005$ (0.5%) or ray exits volume [S3, unofficial].
    *   Composite with solid geometry color if $t_{far}$ was clamped by terrain depth [S2, unofficial].

5.  **Temporal Reprojection (Optional but common):**
    *   Render at reduced resolution (e.g., 0.5x).
    *   Reconstruct world position from depth.
    *   Project previous frame's position using previous view-projection matrix.
    *   Blend current sample with history (e.g., 92% history, 8% current) [S3, unofficial].
    *   Clamp history to 3x3 neighborhood to prevent ghosting on disocclusion [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_clouds(camera_pos, camera_dir, sun_dir, sun_color, time, shape_tex, detail_tex, weather_tex, layer_params):
    # 1. Ray Setup
    ray_origin = camera_pos
    ray_dir = normalize(camera_dir)
    
    # 2. Intersection with Spherical Shell
    oc = ray_origin - vec3(0.0, -layer_params.earth_radius, 0.0)
    b = dot(oc, ray_dir)
    c = dot(oc, oc)
    
    h_outer = b * b - (c - layer_params.r_outer * layer_params.r_outer)
    if h_outer < 0.0:
        return vec4(0.0, 0.0, 0.0, 1.0) # No intersection
    
    s_outer = sqrt(h_outer)
    t_outer_near = -b - s_outer
    t_outer_far = -b + s_outer
    
    h_inner = b * b - (c - layer_params.r_inner * layer_params.r_inner)
    if h_inner > 0.0:
        s_inner = sqrt(h_inner)
    else:
        s_inner = 0.0
        
    t_inner_near = -b - s_inner
    t_inner_far = -b + s_inner
    
    t_near = max(t_outer_near, t_inner_near)
    t_far = min(t_outer_far, t_inner_far)
    
    if t_near > t_far:
        return vec4(0.0, 0.0, 0.0, 1.0)

    # 3. Discretization
    step_size = (t_far - t_near) / layer_params.max_steps
    jitter = get_jitter(time) # Halton sequence
    t_current = t_near + jitter * step_size
    
    transmittance = 1.0
    accumulated_color = vec3(0.0)
    depth_accum = 0.0
    
    # 4. Integration Loop
    for i in range(layer_params.max_steps):
        if transmittance < 0.005:
            break
            
        pos = ray_origin + ray_dir * t_current
        height = length(pos - vec3(0.0, -layer_params.earth_radius, 0.0)) - layer_params.earth_radius
        
        # Density Sampling
        wind_offset = layer_params.wind * time
        shape_pos = (pos + wind_offset) * layer_params.shape_scale
        detail_pos = (pos + wind_offset) * layer_params.detail_scale
        
        n_shape = sample_3d(shape_tex, shape_pos)
        n_detail = sample_3d(detail_tex, detail_pos)
        
        # Weather map lookup for coverage and type
        weather_uv = get_weather_uv(pos)
        weather_data = sample_2d(weather_tex, weather_uv)
        coverage = weather_data.r
        cloud_type = weather_data.g
        
        # Base density from shape noise
        density = n_shape.r # Simplified; actual Nubis uses dimensional profiles
        
        # Erosion
        erosion_weight = pow(1.0 - density, 6.0) # Quintic erosion
        density = max(density - erosion_weight * n_detail.r * (1.0 - coverage), 0.0)
        
        # Vertical Falloff
        falloff = get_vertical_falloff(height, layer_params.base_height, layer_params.thickness)
        density *= falloff
        
        if density <= 0.0:
            t_current += step_size
            continue
            
        # Optical Depth towards Sun
        tau_sun = 0.0
        sun_samples = 6
        for j in range(sun_samples):
            sun_pos = pos + ray_dir * (j + 0.5) * step_size * 0.5 # Approximate direction
            # Note: In practice, sample along sun_dir, not ray_dir
            sun_sample_pos = pos + normalize(sun_dir) * (j + 0.5) * step_size
            tau_sun += sample_3d(shape_tex, sun_sample_pos).r * step_size
            
        # Lighting
        cos_angle = dot(normalize(ray_dir), normalize(sun_dir))
        phase = heney_greenstein(cos_angle, layer_params.anisotropy)
        
        # Multiple Scattering Approximation
        ms_light = compute_multiple_scattering(tau_sun, phase, layer_params)
        
        # Ambient
        ambient_top = vec3(0.62, 0.74, 0.95)
        ambient_bottom = vec3(0.36, 0.42, 0.52)
        ambient = mix(ambient_bottom, ambient_top, smoothstep(0.0, layer_params.thickness, height))
        
        # Powder Term
        powder = 1.0 - exp(-tau_sun * 2.0) # Darken back-lit
        
        # In-Scattering
        in_scatter = ms_light * sun_color * phase * (1.0 - powder) + ambient * 0.1
        accumulated_color += in_scatter * transmittance * step_size
        
        # Transmittance Update
        transmittance *= exp(-density * step_size)
        
        # Depth Accumulation
        depth_accum += t_current * transmittance * step_size
        
        t_current += step_size
        
    # 5. Output
    mean_depth = depth_accum / (1.0 - transmittance + 1e-6)
    return vec4(accumulated_color, transmittance), mean_depth
```

### Parameters
*   `shape_noise_scale`: Default 6.4 km. Controls cloud size [S1, unofficial].
*   `max_steps`: Default 128. Maximum number of march steps [S3, unofficial].
*   `anisotropy`: Parameter for Henyey-Greenstein phase function [S3, unofficial].
*   `wind`: Vec2 wind vector for scrolling noise [S3, unofficial].
*   `coverage`: Scales cloud fraction [S1, unofficial].
*   `base_height`: Altitude of cloud base [S3, unofficial].
*   `thickness`: Thickness of cloud layer [S3, unofficial].

### Edge Cases and Pitfalls
*   **Shadow Map Basis Error:** If multiple non-parallel suns exist, using a single shadow map traced along the first light's basis causes displaced shadows for other lights. Each light should march its own transmittance or use a shared map in a consistent basis [S1, unofficial].
*   **Reprojection Ghosting:** Using mean depth for reprojection can cause ghosting when thin near clouds overlap far clouds. Clamping history to a 3x3 neighborhood mitigates this [S3, unofficial].
*   **History Validity:** History textures must be checked for identity. New views, reallocations, or resizes invalidate history [S1, unofficial].
*   **Viewport Mismatch:** Depth texture reads must account for viewport origin to avoid tracing the wrong part of the view [S1, unofficial].

## Subtopics

### [Volumetric Clouds from Base Shape (Nubis/Frostbite Hybrid)](9609-volumetric-clouds-from-base-shape-nubis-frostbite-hybrid.md)
Renders clouds as a spherical shell above a base shape, combining Nubis density with Frostbite light transport. Chosen when clouds must conform to terrain without 2D tearing.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of pixels and $M$ is the number of steps per ray. With temporal reprojection, effective cost is reduced by the resolution scale factor squared.
*   **Memory Complexity:** $O(T_{shape} + T_{detail} + T_{weather} + T_{history})$. Requires 3D textures for noise and framebuffers for temporal history.
*   **Performance:** On CPU rasterizer (Mesa llvmpipe) at 640x360, full-res 128 steps takes ~927 ms. Half-res with temporal reprojection takes ~266 ms (29% of full-res) [S3, unofficial].
*   **Strengths:** Real-time performance, physically plausible lighting, dynamic movement, works at any camera position relative to the layer.
*   **Limitations:** Approximation of multiple scattering (flatter lit side than path-traced), single layer support (in basic implementations), hand-tuned sun color at low elevation [S3, unofficial].

## Background
The technique originates from real-time game development, specifically inspired by *Horizon: Zero Dawn* (Schneider 2015) and *Frostbite* engine techniques (Hillaire 2016, 2020). It was further refined in *Nubis Evolved* (Schneider 2022) and *Nubis³* (Schneider 2023). Implementations exist in various engines and open-source projects, including Bevy, Unity, and standalone OpenGL renderers [S1, S2, S3, S4, unofficial].

## Sources
[S1] WIP: Nubis-based volumetric clouds, https://github.com/mate-h/bevy/pull/26, fetched 2026-10-27 (unofficial)
[S2] Depersonalizc/volumetric-clouds, https://github.com/Depersonalizc/volumetric-clouds, fetched 2026-10-27 (unofficial)
[S3] harkawataryan/Volumetric-Cloud-Renderer, https://github.com/harkawataryan/Volumetric-Cloud-Renderer, fetched 2026-10-27 (unofficial)
[S4] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
