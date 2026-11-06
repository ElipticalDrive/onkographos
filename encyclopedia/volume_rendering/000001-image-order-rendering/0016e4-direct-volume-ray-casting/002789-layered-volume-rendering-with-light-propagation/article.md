# Layered Volume Rendering with Light Propagation

Also called: Layer-based volumetric ray marching, Plane-sweep volume rendering

## Overview
Layered Volume Rendering with Light Propagation is a compute-shader-based technique that renders volumetric data by processing a stack of parallel planes (layers) rather than individual pixels or rays [S1, unofficial]. It propagates light and view rays across these layers, accumulating color, alpha, and material properties while applying refraction gradients and transfer functions [S1, unofficial]. It sets itself apart from standard direct volume ray casting by decoupling the light transport calculation from the view ray accumulation into a separate propagation pass, allowing for efficient screen-space approximation of light scattering and refraction effects across the volume layers [S1, unofficial].

## How it works
The method operates in a compute shader, processing a stack of $N$ parallel planes. It maintains state buffers for light propagation and view ray accumulation, updating them layer by layer from front to back.

### Inputs and Outputs
**Inputs:**
*   `ubo`: Uniform buffer object containing rendering parameters:
    *   `plane_pixel_size`: Pixel size of the rendering planes in world units [S1, unofficial].
    *   `distance_between_planes`: Constant Z-distance between consecutive layers [S1, unofficial].
    *   `lightVSPos`: View-space position of the light source [S1, unofficial].
    *   `light_color`: View-space color/intensity of the light source [S1, unofficial].
    *   `plane_count`: Total number of layers [S1, unofficial].
*   `volume_density`: 3D texture or stack of 2D textures representing scalar volume density [S1, unofficial].
*   `volume_transfer`: 2D texture mapping `vec2(density_in, density_prev)` to `vec4(alpha_out)` [S1, unofficial].
*   `medium_transfer`: 2D texture mapping `vec2(density_in, density_prev)` to `vec4(tint_out)` [S1, unofficial].
*   Input buffers (from previous layer or initialization):
    *   `ldb_in`: Light direction buffer [S1, unofficial].
    *   `lb_in`: Light color/intensity buffer [S1, unofficial].
    *   `vpb_in`: View position buffer [S1, unofficial].
    *   `vdb_in`: View direction buffer [S1, unofficial].
    *   `cb_in`: Accumulated color buffer [S1, unofficial].
    *   `mb_in`: Accumulated material buffer [S1, unofficial].

**Outputs:**
*   `lb_out`: Updated light color buffer [S1, unofficial].
*   `ldb_out`: Updated light direction buffer [S1, unofficial].
*   `cb_out`: Final accumulated color and alpha buffer [S1, unofficial].
*   `mb_out`: Final accumulated material buffer [S1, unofficial].
*   `vpb_out`: Next view position for the subsequent layer [S1, unofficial].
*   `vdb_out`: Next view direction for the subsequent layer [S1, unofficial].

### Data Structures and Layout
All buffers are accessed via `imageLoad` and `imageStore` using `ivec2` pixel coordinates corresponding to the screen resolution [S1, unofficial].
*   `lb_in/out`: `vec3` (light intensity/color) [S1, unofficial].
*   `ldb_in/out`: `vec3` (light direction) [S1, unofficial].
*   `vpb_in/out`: `vec2` or `vec3` (view position, likely 2D UV or 3D world position projected) [S1, unofficial].
*   `vdb_in/out`: `vec3` (view direction) [S1, unofficial].
*   `cb_in/out`: `vec4` (color + alpha) [S1, unofficial].
*   `mb_in/out`: `vec4` (material properties) [S1, unofficial].

### Steps

1.  **Initialization**:
    *   Compute UV coordinates and pixel coordinates from `gl_GlobalInvocationID` [S1, unofficial].
    *   Calculate the current layer's Z-coordinate:
        $$ z_{current} = z_{front} - \text{layer\_index} \times \text{distance\_between\_planes} $$
        where $z_{front}$ is the Z-coordinate of the front-most plane (`ubo.middle_of_front_plane_vs.z`) [S1, unofficial].

2.  **Light Propagation Pass**:
    This pass updates the light state as it propagates through the volume layers.
    *   **Intersection**: Sample the current light direction $\mathbf{L}_{dir}$ from `ldb_in` at the current UV. Intersect a ray starting at $(0.5, 0.5, z_{current} + \text{distance})$ with direction $-\mathbf{L}_{dir}$ to find the hit point $\mathbf{P}_{prev}$ on the previous plane (closer to the light) [S1, unofficial].
    *   **Sampling & Averaging**:
        *   Sample light intensity $I_{prev}$ from `lb_in` at $\mathbf{P}_{prev}.xy$ [S1, unofficial].
        *   Sample light direction $\mathbf{L}_{prev}$ from `ldb_in` at $\mathbf{P}_{prev}.xy$.
        *   Apply 4-tap averaging to $\mathbf{L}_{prev}$: center weight 0.5, four neighbors weight 0.125 each [S1, unofficial].
        *   Normalize $\mathbf{L}_{prev}$. If $\|\mathbf{L}_{prev}\| < 10^{-6}$, set $\mathbf{L}_{prev} = (0, 0, 1)$ to prevent NaN propagation [S1, unofficial].
    *   **Irradiance Scaling**:
        *   Compute screen-space area factors: $S_{current} = |\frac{\partial u}{\partial x}| \cdot |\frac{\partial v}{\partial y}|$ and $S_{prev} = |\frac{\partial u_{prev}}{\partial x}| \cdot |\frac{\partial v_{prev}}{\partial y}|$ using `dFdx` and `dFdy` [S1, unofficial].
        *   Compute irradiance scaling factor $I_{scale}$:
            $$ I_{scale} = \begin{cases} S_{prev} / S_{current} & \text{if } S_{current} > 10^{-8} \\ 1.0 & \text{otherwise} \end{cases} $$
            This prevents division by zero [S1, unofficial].
    *   **Transfer Function Application**:
        *   Convert current UV and $\mathbf{P}_{prev}.xy$ to world positions at $z_{current}$ and $z_{current} + \text{distance}$, denoted $\mathbf{P}_{ws}$ and $\mathbf{P}_{prev\_ws}$ [S1, unofficial].
        *   Sample volume density $D_{current}$ at $\mathbf{P}_{ws}$ and $D_{prev}$ at $\mathbf{P}_{prev\_ws}$ [S1, unofficial].
        *   Sample `volume_transfer` with `vec2(D_current, D_prev)` to get opacity $\alpha$ [S1, unofficial].
        *   Sample `medium_transfer` with `vec2(D_current, D_prev)` to get tint color $\mathbf{T}$ via `mix(vec3(1.0), m_data.rgb, m_data.a)` [S1, unofficial].
        *   Compute updated light intensity:
            $$ \mathbf{L}_{out} = \mathbf{L}_{prev} \times I_{scale} \times (1.0 - \alpha) \times \mathbf{T} $$
        *   Clamp $\mathbf{L}_{out}$ to a minimum value of `vec3(20.0)` [S1, unofficial].
        *   Store $\mathbf{L}_{out}$ to `lb_out` [S1, unofficial].
    *   **Light Refraction Update**:
        *   Compute midpoint $\mathbf{M}_{ws} = (\mathbf{P}_{ws} + \mathbf{P}_{prev\_ws}) \times 0.5$ [S1, unofficial].
        *   Get refraction gradient $\nabla \mathbf{L}$ at $\mathbf{M}_{ws}$ (derived from volume density gradients) [S1, unofficial].
        *   Update light direction:
            $$ \mathbf{L}_{next} = \mathbf{L}_{prev} + \nabla \mathbf{L} \times \text{distance\_between\_planes} $$
        *   Normalize $\mathbf{L}_{next}$ and store to `ldb_out` [S1, unofficial].

3.  **Ray Propagation Pass**:
    This pass accumulates the view ray through the volume, using the pre-computed light state.
    *   **Setup**:
        *   Sample view position $\mathbf{V}_{pos}$ from `vpb_in` and view direction $\mathbf{V}_{dir}$ from `vdb_in` [S1, unofficial].
        *   Convert $\mathbf{V}_{pos}$ to world space [S1, unofficial].
        *   Sample previous accumulated color $\mathbf{C}_{prev}$ and alpha $A_{prev}$ from `cb_in` [S1, unofficial].
        *   Sample previous material $\mathbf{M}_{prev}$ from `mb_in` [S1, unofficial].
        *   Sample light intensity $\mathbf{L}_{id}$ from `lb_in` (using current UV or propagated coordinates) [S1, unofficial].
    *   **Intersection**:
        *   Intersect a ray from $\mathbf{V}_{pos}$ with direction $-\mathbf{V}_{dir}$ to the plane at $z_{current} + \text{distance}$ to get the previous view position $\mathbf{V}_{prev}$ [S1, unofficial].
        *   Convert $\mathbf{V}_{prev}$ to world space $\mathbf{V}_{prev\_ws}$ [S1, unofficial].
    *   **Shading**:
        *   Compute volume gradient $\mathbf{N}$ at $\mathbf{V}_{ws} + 0.5$ [S1, unofficial].
        *   Define view vector $\mathbf{V} = \text{normalize}(-\mathbf{V}_{dir})$ [S1, unofficial].
        *   Define light vector $\mathbf{L}_{dir} = \text{normalize}(\text{ubo.lightVSPos} - \mathbf{V}_{pos})$ [S1, unofficial].
        *   Compute half-vector $\mathbf{H} = \text{normalize}(\mathbf{L}_{dir} + \mathbf{V})$ [S1, unofficial].
        *   Compute specular term: $\text{spec} = \max(\mathbf{N} \cdot \mathbf{H}, 0.0)^{128.0}$ [S1, unofficial].
        *   Compute specular intensity: $\mathbf{I}_s = \text{ubo.light\_color} \times \text{spec} \times 0.05$ [S1, unofficial].
        *   Compute material mask: $\mathbf{M}_{mask} = \text{smoothstep}(0.0, 0.05, D_{current})$ [S1, unofficial].
    *   **Accumulation**:
        *   The opacity $\alpha$ and tint $\mathbf{T}$ are obtained from the Light Propagation step (using the same density samples) [S1, unofficial].
        *   Update accumulated color:
            $$ \mathbf{C}_{current} = \mathbf{C}_{prev} + (1.0 - A_{prev}) \times \mathbf{M}_{prev} \times (\alpha \times \mathbf{T} \times \mathbf{L}_{id} + \mathbf{I}_s \times \mathbf{M}_{mask}) $$
            [S1, unofficial].
        *   Update accumulated alpha:
            $$ A_{current} = A_{prev} + (1.0 - A_{prev}) \times \alpha $$
            [S1, unofficial].
        *   Update accumulated material:
            $$ \mathbf{M}_{current} = \mathbf{M}_{prev} \times (1.0 - \mathbf{M}_{prev}.w) + \mathbf{M}_{prev} \times \mathbf{M}_{prev}.xyz \times \mathbf{M}_{prev}.w $$
            [S1, unofficial].
    *   **Refraction & Next Step**:
        *   Compute midpoint $\mathbf{M}_{vp} = (\mathbf{V}_{ws} + \mathbf{V}_{prev\_ws}) \times 0.5 + 0.5$ [S1, unofficial].
        *   Get refraction gradient $\nabla \mathbf{V}$ at $\mathbf{M}_{vp}$ [S1, unofficial].
        *   Update view direction:
            $$ \mathbf{V}_{next} = \mathbf{V}_{dir} + \nabla \mathbf{V} \times \text{distance\_between\_planes} $$
        *   Normalize $\mathbf{V}_{next}$ [S1, unofficial].
        *   Intersect a ray from $\mathbf{V}_{pos}$ with $\mathbf{V}_{next}$ to the plane at $z_{current} - \text{distance}$ to get $\mathbf{V}_{next\_pos}$ [S1, unofficial].
    *   **Background Replacement (Last Layer)**:
        *   If $\text{layer\_index} == \text{plane\_count} - 1$:
            *   Generate checkerboard background:
                $$ \text{scale} = 16 $$
                $$ \text{check\_uv} = \mathbf{V}_{next\_pos}.xy \times \text{scale} $$
                $$ f = \text{mod}(\lfloor \text{check\_uv}.x \rfloor + \lfloor \text{check\_uv}.y \rfloor, 2.0) $$
                $$ \mathbf{C}_{bg} = \text{mix}(\text{vec3}(0.2), \text{vec3}(0.5), f) $$
                [S1, unofficial].
            *   Blend background with accumulated color:
                $$ \mathbf{C}_{final} = \text{mix}(\mathbf{C}_{current}, \mathbf{C}_{bg}, 1.0 - A_{current}) $$
                [S1, unofficial].
            *   Set $A_{current} = 1.0$ [S1, unofficial].
    *   **Output Storage**:
        *   Store $\mathbf{C}_{final}$ and $A_{current}$ to `cb_out` [S1, unofficial].
        *   Store $\mathbf{M}_{current}$ to `mb_out` [S1, unofficial].
        *   Store $\mathbf{V}_{next\_pos}$ to `vpb_out` [S1, unofficial].
        *   Store $\mathbf{V}_{next}$ to `vdb_out` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Layered Volume Rendering with Light Propagation
# Assumes compute shader invocation context

def process_layer(layer_index, plane_count, distance_between_planes, front_plane_z, 
                  plane_pixel_size, light_vs_pos, light_color,
                  volume_density, volume_transfer, medium_transfer,
                  ldb_in, lb_in, vpb_in, vdb_in, cb_in, mb_in):
    
    # 1. Initialization
    uv = compute_uv_from_global_id()
    pixel_coords = compute_pixel_coords_from_global_id()
    current_z = front_plane_z - layer_index * distance_between_planes
    
    # 2. Light Propagation
    # Sample current light direction
    ld_current = image_load(ldb_in, pixel_coords)
    
    # Intersect ray from (0.5, 0.5, current_z + distance) with -ld_current
    # to find hit point on previous plane
    lp_prev = intersect_ray(vec3(0.5, 0.5, current_z + distance_between_planes), -ld_current, current_z + distance_between_planes)
    
    # Sample and average light direction from previous plane
    ld_prev = image_load(ldb_in, lp_prev.xy)
    ld_prev = average_4_tap(ld_prev, lp_prev.xy) # Center 0.5, neighbors 0.125
    if length(ld_prev) < 1e-6:
        ld_prev = vec3(0, 0, 1)
    
    # Irradiance scaling
    s_current = abs(dFdx(uv.x)) * abs(dFdy(uv.y))
    s_prev = abs(dFdx(lp_prev.x)) * abs(dFdy(lp_prev.y))
    if s_current > 1e-8:
        irradiance_scale = s_prev / s_current
    else:
        irradiance_scale = 1.0
        
    # Transfer function application
    lp_ws = uv_to_world(uv, current_z)
    lp_prev_ws = uv_to_world(lp_prev.xy, current_z + distance_between_planes)
    
    density_current = sample_volume(volume_density, lp_ws)
    density_prev = sample_volume(volume_density, lp_prev_ws)
    
    tf_input = vec2(density_current, density_prev)
    alpha = sample_texture(volume_transfer, tf_input).a # Or specific component
    tint = sample_texture(medium_transfer, tf_input)
    tint = mix(vec3(1.0), tint.rgb, tint.a)
    
    # Light intensity update
    l_prev_intensity = image_load(lb_in, lp_prev.xy)
    l_out = l_prev_intensity * irradiance_scale * (1.0 - alpha) * tint
    l_out = max(l_out, vec3(20.0))
    
    # Light refraction update
    mid_lp_ws = (lp_ws + lp_prev_ws) * 0.5
    ref_grad_lp = compute_refraction_gradient(volume_density, mid_lp_ws)
    ld_next = normalize(ld_prev + ref_grad_lp * distance_between_planes)
    
    # Store light outputs
    image_store(lb_out, pixel_coords, l_out)
    image_store(ldb_out, pixel_coords, ld_next)
    
    # 3. Ray Propagation
    # Setup
    vp_current = image_load(vpb_in, pixel_coords)
    vd_current = image_load(vdb_in, pixel_coords)
    vp_ws = uv_to_world(vp_current, current_z) # Assuming vp_current is UV-like
    
    c_prev = image_load(cb_in, pixel_coords).rgb
    a_prev = image_load(cb_in, pixel_coords).a
    m_prev = image_load(mb_in, pixel_coords)
    l_id = image_load(lb_in, pixel_coords) # Light intensity at current layer
    
    # Intersection
    vp_prev = intersect_ray(vp_ws, -vd_current, current_z + distance_between_planes)
    vp_prev_ws = uv_to_world(vp_prev, current_z + distance_between_planes)
    
    # Shading
    n = compute_volume_gradient(volume_density, vp_ws + 0.5)
    v = normalize(-vd_current)
    l_dir = normalize(light_vs_pos - vp_ws) # Light direction from surface to light
    h = normalize(l_dir + v)
    spec = pow(max(dot(n, h), 0.0), 128.0)
    ispec = light_color * spec * 0.05
    material_mask = smoothstep(0.0, 0.05, density_current)
    
    # Accumulation
    # Note: alpha and tint are from Light Propagation step, re-sampled or passed
    c_current = c_prev + (1.0 - a_prev) * m_prev * (alpha * tint * l_id + ispec * material_mask)
    a_current = a_prev + (1.0 - a_prev) * alpha
    m_current = m_prev * (1.0 - m_prev.w) + m_prev * m_prev.xyz * m_prev.w
    
    # Refraction & Next Step
    mid_vp_ws = (vp_ws + vp_prev_ws) * 0.5 + 0.5
    ref_grad_vp = compute_refraction_gradient(volume_density, mid_vp_ws)
    vd_next = normalize(vd_current + ref_grad_vp * distance_between_planes)
    vp_next = intersect_ray(vp_ws, vd_next, current_z - distance_between_planes)
    
    # Background Replacement (Last Layer)
    if layer_index == plane_count - 1:
        scale = 16
        check_uv = vp_next.xy * scale
        f = mod(floor(check_uv.x) + floor(check_uv.y), 2.0)
        bg = mix(vec3(0.2), vec3(0.5), f)
        c_final = mix(c_current, bg, 1.0 - a_current)
        a_current = 1.0
    else:
        c_final = c_current
        
    # Output Storage
    image_store(cb_out, pixel_coords, vec4(c_final, a_current))
    image_store(mb_out, pixel_coords, m_current)
    image_store(vpb_out, pixel_coords, vp_next)
    image_store(vdb_out, pixel_coords, vd_next)

def average_4_tap(vec, uv):
    # Center
    val = vec * 0.5
    # Neighbors (assuming standard 4-tap in UV space)
    # Note: Exact neighbor offsets depend on implementation details not fully specified
    # This is a placeholder for the averaging logic described
    val += sample_neighbor(uv, offset=(1,0)) * 0.125
    val += sample_neighbor(uv, offset=(-1,0)) * 0.125
    val += sample_neighbor(uv, offset=(0,1)) * 0.125
    val += sample_neighbor(uv, offset=(0,-1)) * 0.125
    return val

def intersect_ray(origin, direction, plane_z):
    # Ray: P = origin + t * direction
    # Plane: P.z = plane_z
    # t = (plane_z - origin.z) / direction.z
    t = (plane_z - origin.z) / direction.z
    return origin + t * direction

def uv_to_world(uv, z):
    # Convert UV (0-1) to world coordinates at height z
    # Assuming standard camera frustum mapping or similar
    # This is a placeholder; exact mapping depends on camera setup
    return vec3(uv * plane_pixel_size * resolution, z)

def compute_refraction_gradient(volume_density, point):
    # Compute gradient of volume density at point
    # Using central differences or Sobel
    return gradient(volume_density, point)
```

### Parameters
*   `ubo.plane_pixel_size`: Pixel size of the rendering planes in world units [S1, unofficial].
*   `ubo.distance_between_planes`: Constant Z-distance between consecutive layers [S1, unofficial].
*   `ubo.lightVSPos`: View-space position of the light source [S1, unofficial].
*   `ubo.light_color`: View-space color/intensity of the light source [S1, unofficial].
*   `ubo.plane_count`: Total number of layers [S1, unofficial].

### Edge Cases and Pitfalls
*   **Zero-length light direction**: If the normalized light direction $\mathbf{L}_{prev}$ has length $< 10^{-6}$ after averaging, it is replaced with `vec3(0,0,1)` to prevent NaN propagation or division by zero in subsequent shading [S1, unofficial].
*   **Division by zero in irradiance scaling**: If the screen-space area factor $S_{current} < 10^{-8}$, the irradiance scaling factor $I_{scale}$ is clamped to 1.0 to avoid NaN or infinite values [S1, unofficial].
*   **Layer boundary handling**: On the last layer (`layer_index == plane_count - 1`), the accumulated alpha is forced to 1.0 and a checkerboard background is blended in to prevent transparent edges or missing geometry [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The method processes each layer once, with constant-time operations per pixel per layer. Total time is $O(N \cdot W \cdot H)$, where $N$ is the number of layers, and $W, H$ are image dimensions [S1, unofficial].
*   **Memory Complexity**: Requires storage for multiple buffers (light direction, light color, view position, view direction, color, material) for each layer or ping-pong buffers for two layers [S1, unofficial].
*   **Strengths**: Efficient screen-space approximation of light scattering and refraction; suitable for real-time rendering on GPUs via compute shaders [S1, unofficial].
*   **Limitations**: Approximates light transport within the layer stack; may suffer from artifacts at layer boundaries or with large inter-layer distances; requires careful tuning of refraction gradients [S1, unofficial].
*   **When to use**: Real-time volumetric rendering where approximate light transport effects are desired and full path tracing is too expensive [S1, unofficial].
*   **When not to use**: When physically accurate light transport is required, or when the volume has complex internal structures that are not well-approximated by planar layers [S1, unofficial].

## Background
The method is described in the context of a Vulkan-based volume rendering implementation, specifically in the `vis-magnus-2018` repository [S1, unofficial]. It represents a compute-shader-based approach to layered volume rendering, focusing on propagating light and view rays across parallel planes [S1, unofficial].

## Sources
[S1] GitThirteen/vis-magnus-2018, https://github.com/GitThirteen/vis-magnus-2018, fetched 2026-10-26 (unofficial)
