# WebGL Path Tracing Volume Rendering

Also called: Interactive Voxel Volume Rendering via WebGL 2.0

## Overview
An interactive voxel volume renderer that runs in modern web browsers via WebGL 2.0, rendering 3D scalar fields by casting rays from the camera through the volume, accumulating color and opacity, and applying lighting and shadowing [S1, unofficial]. It distinguishes itself from standard direct volume ray casting by implementing a path-tracing-like shading model (diffuse, specular, and soft shadows) within the fragment shader, while using a linear search through control points for transfer functions to support arbitrary non-uniform sampling [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**:
    *   `volume`: `highp sampler3D` containing the scalar field.
    *   `volume_dims`: `ivec3` defining the dimensions of the volume texture.
    *   `vray_dir`: `vec3` ray direction in world space.
    *   `transformed_eye`: `vec3` camera origin in world space.
    *   `lightPosition`: `vec3` position of the light source.
    *   `windowLevel`, `windowWidth`: Floats defining the DICOM-like windowing for value rescaling.
    *   `rescaleSlope`, `rescaleIntercept`: Floats for linear transformation of raw values.
    *   `dt_scale`: Float scaling factor for the ray marching step size.
    *   `opacityMapPositions`, `opacityMapSamples`: `highp sampler2D` textures storing control points for opacity mapping.
    *   `colorMapPositions`, `colorMapSamples`: `highp sampler2D` textures storing control points for color mapping.
    *   `numberOfOpacitySamples`, `numberOfColorSamples`: Integers defining the number of control points.
*   **Outputs**:
    *   `color`: `vec4` final accumulated color and opacity for the pixel.

**Data Structures**
The transfer functions are stored in 2D textures where one dimension represents the position (value) and the other represents the sample (opacity or color). The renderer performs a linear search through these control points rather than using texture interpolation, allowing for non-uniform spacing of control points [S1, unofficial].

**Algorithm Steps**

1.  **Ray-Box Intersection**:
    Compute the inverse ray direction `invDir = 1.0 / ray_dir`.
    Calculate temporary bounds:
    $$ t_{min\_tmp} = (boxMin - orig) \cdot invDir $$
    $$ t_{max\_tmp} = (boxMax - orig) \cdot invDir $$
    Determine entry and exit times:
    $$ t_{min} = \min(t_{min\_tmp}, t_{max\_tmp}) $$
    $$ t_{max} = \max(t_{min\_tmp}, t_{max\_tmp}) $$
    $$ t_0 = \max(t_{min}.x, \max(t_{min}.y, t_{min}.z)) $$
    $$ t_1 = \min(t_{max}.x, \min(t_{max}.y, t_{max}.z)) $$
    If $t_0 > t_1$, the ray does not intersect the volume; discard the fragment [S1, unofficial].

2.  **Step Size Calculation**:
    Compute the step size based on the volume dimensions and ray direction to ensure consistent voxel traversal:
    $$ dt_{vec} = \frac{1.0}{vec3(volume\_dims) \cdot |ray\_dir|} $$
    $$ dt = dt\_scale \cdot \min(dt_{vec}.x, \min(dt_{vec}.y, dt_{vec}.z)) $$
    [S1, unofficial]

3.  **Ray Origin Offset**:
    Generate a random offset to reduce aliasing using a Wang hash:
    $$ seed = (seed \oplus 61) \oplus (seed \gg 16) $$
    $$ seed = seed \cdot 9 $$
    $$ seed = seed \oplus (seed \gg 4) $$
    $$ seed = seed \cdot 0x27d4eb2d $$
    $$ seed = seed \oplus (seed \gg 15) $$
    $$ offset = \frac{seed \pmod{2147483647}}{2147483647} $$
    Initial position:
    $$ p = transformed\_eye + (t_0 + offset \cdot dt) \cdot ray\_dir $$
    [S1, unofficial]

4.  **Ray Marching Loop**:
    Initialize accumulated color `voxelColor = vec4(0.0)`.
    Loop $t$ from $t_0$ to $t_1$ with step $dt$:
    a.  **Sample Volume**: Retrieve raw value $V$ from `volume` at position $p$.
    b.  **Rescale Value**:
        $$ V_{rescaled} = rescaleSlope \cdot V + rescaleIntercept $$
        $$ V_{windowed} = \frac{V_{rescaled} - (windowLevel - windowWidth/2.0)}{windowWidth} $$
    c.  **Transfer Function**:
        *   **Opacity**: Find interval $[opacityMapPositions(i-1), opacityMapPositions(i)]$ containing $V_{windowed}$. Interpolate between $opacityMapSamples(i-1)$ and $opacityMapSamples(i)$. Clamp result to $[0, 1]$. Let this be $\alpha_{raw}$.
        *   **Color**: Similarly, interpolate color $\mathbf{c}_{raw}$ from `colorMapSamples`.
    d.  **Beer-Lambert Approximation**:
        $$ \alpha_{step} = 1.0 - (1.0 - \alpha_{raw})^{dt\_scale} $$
        $$ \mathbf{c}_{step} = \mathbf{c}_{raw} \cdot \alpha_{step} $$
    e.  **Compositing**:
        $$ \mathbf{voxelColor}.rgb \leftarrow \mathbf{voxelColor}.rgb + (1.0 - \mathbf{voxelColor}.a) \cdot \mathbf{c}_{step} $$
        $$ \mathbf{voxelColor}.a \leftarrow \mathbf{voxelColor}.a + (1.0 - \mathbf{voxelColor}.a) \cdot \alpha_{step} $$
    f.  **Early Termination**: If $\mathbf{voxelColor}.a \ge 0.95$, break loop.
    g.  **Advance**: $p \leftarrow p + ray\_dir \cdot dt$
    [S1, unofficial]

5.  **Lighting and Shading**:
    If $\mathbf{voxelColor}.a < 0.20$, lighting is skipped (shadow = 1.0). Otherwise:
    a.  **Normal Estimation**:
        Compute voxel size $voxelSize = 1.0 / vec3(volume\_dims)$.
        Estimate gradient $\nabla V$ using central differences on the opacity-mapped value at $p$:
        $$ \nabla V_x = \frac{Opacity(p + (voxelSize.x, 0, 0)) - Opacity(p - (voxelSize.x, 0, 0))}{2 \cdot voxelSize.x} $$
        (Similarly for y and z).
        $$ voxelNormal = normalize(\nabla V) $$
    b.  **Diffuse Lighting**:
        $$ lightDirection = normalize(lightPosition - p) $$
        $$ intensity = \max(dot(voxelNormal, lightDirection), 0.0) $$
        $$ diffuseIntensity = intensity \cdot 0.6 $$
    c.  **Specular Lighting**:
        $$ h = normalize(lightDirection + (-transformed\_eye)) $$
        $$ intSpec = \max(dot(h, voxelNormal), 0.0) $$
        $$ spec = 0.5 \cdot intSpec^{8.0} $$
    d.  **Shadow Calculation**:
        Compute `softShadow(p)` by casting a shadow ray (see Step 6).
        $$ shadow = clamp(softShadow(p) \cdot 2.0 + 0.1, 0.0, 1.0) $$
    e.  **Final Color**:
        $$ lightColor = shadow \cdot (diffuseIntensity + spec + 0.4) $$
        $$ \mathbf{voxelColor}.rgb \leftarrow \mathbf{voxelColor}.rgb \cdot lightColor $$
    [S1, unofficial]

6.  **Soft Shadow Calculation**:
    Cast a ray from $p$ towards $lightPosition$.
    $$ voxelToLight = lightPosition - p $$
    $$ d = normalize(voxelToLight) $$
    $$ o = p - 1000.0 \cdot d $$
    Intersect ray $(o, d)$ with volume box to get $[t_{start}, t_{end}]$.
    Sample opacity along the shadow ray from $t_{start}$ to $t_{end}$ with step $\delta = \min(voxelSize) / samplesPerVoxel$ (where $samplesPerVoxel = 1.0$).
    Accumulate integrated opacity:
    $$ integrated = 0 $$
    For each sample $val$:
    $$ integrated \leftarrow integrated + \frac{val}{samplesPerVoxel \cdot 4.0} $$
    If $integrated > 0.95$, break.
    Return $clamp(integrated, 0.0, 1.0)$.
    [S1, unofficial]

7.  **SRGB Conversion**:
    Convert final linear color to sRGB for display:
    If $x \le 0.0031308$, return $12.92 \cdot x$.
    Else return $1.055 \cdot x^{1/2.4} - 0.055$.
    [S1, unofficial]

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_fragment(vray_dir, transformed_eye, volume, volume_dims, dt_scale, 
                    light_position, window_level, window_width, rescale_slope, 
                    rescale_intercept, opacity_map_positions, opacity_map_samples,
                    color_map_positions, color_map_samples, num_opacity_samples, 
                    num_color_samples):
    # 1. Ray-Box Intersection
    inv_dir = 1.0 / vray_dir
    t_min_tmp = (box_min - transformed_eye) * inv_dir
    t_max_tmp = (box_max - transformed_eye) * inv_dir
    t_min = min(t_min_tmp, t_max_tmp)
    t_max = max(t_min_tmp, t_max_tmp)
    t0 = max(t_min.x, max(t_min.y, t_min.z))
    t1 = min(t_max.x, min(t_max.y, t_max.z))
    
    if t0 > t1:
        return vec4(0.0) # Discard fragment

    # 2. Step Size
    dt_vec = 1.0 / (vec3(volume_dims) * abs(vray_dir))
    dt = dt_scale * min(dt_vec.x, min(dt_vec.y, dt_vec.z))

    # 3. Ray Origin Offset
    seed = wang_hash(int(gl_FragCoord.x + 1024.0 * gl_FragCoord.y))
    offset = float(seed % 2147483647) / 2147483647.0
    p = transformed_eye + (t0 + offset * dt) * vray_dir

    # 4. Ray Marching
    voxel_color = vec4(0.0)
    t = t0
    while t < t1:
        # Sample Volume
        val = sample_volume(volume, p)
        
        # Rescale and Window
        rescaled_val = rescale_slope * val + rescale_intercept
        windowed_val = (rescaled_val - (window_level - window_width / 2.0)) / window_width
        
        # Transfer Function (Linear Search)
        alpha_raw = lookup_transfer(windowed_val, opacity_map_positions, opacity_map_samples, num_opacity_samples)
        color_raw = lookup_transfer(windowed_val, color_map_positions, color_map_samples, num_color_samples)
        
        # Beer-Lambert
        alpha_step = 1.0 - pow(1.0 - alpha_raw, dt_scale)
        color_step = color_raw * alpha_step
        
        # Compositing
        voxel_color.rgb += (1.0 - voxel_color.a) * color_step.rgb
        voxel_color.a += (1.0 - voxel_color.a) * alpha_step
        
        if voxel_color.a >= 0.95:
            break
            
        p += vray_dir * dt
        t += dt

    # 5. Lighting
    if voxel_color.a >= 0.20:
        # Normal Estimation
        voxel_size = 1.0 / vec3(volume_dims)
        grad = compute_gradient_opacity(p, voxel_size, opacity_map_positions, opacity_map_samples, num_opacity_samples, window_level, window_width, rescale_slope, rescale_intercept)
        voxel_normal = normalize(grad)
        
        # Diffuse
        light_dir = normalize(light_position - p)
        intensity = max(dot(voxel_normal, light_dir), 0.0)
        diffuse = intensity * 0.6
        
        # Specular
        h = normalize(light_dir + (-transformed_eye))
        int_spec = max(dot(h, voxel_normal), 0.0)
        spec = 0.5 * pow(int_spec, 8.0)
        
        # Shadow
        shadow_val = soft_shadow(p, light_position, volume, voxel_size)
        shadow = clamp(shadow_val * 2.0 + 0.1, 0.0, 1.0)
        
        light_color = shadow * (diffuse + spec + 0.4)
        voxel_color.rgb *= light_color

    # 7. SRGB Conversion
    voxel_color.rgb = srgb_convert(voxel_color.rgb)
    
    return voxel_color

def lookup_transfer(value, positions, samples, num_samples):
    # Linear search for interval
    for i in range(1, num_samples):
        if positions[i] >= value:
            # Interpolate between i-1 and i
            t = (value - positions[i-1]) / (positions[i] - positions[i-1])
            return samples[i-1] + t * (samples[i] - samples[i-1])
    return samples[num_samples-1]

def soft_shadow(p, light_pos, volume, voxel_size):
    voxel_to_light = light_pos - p
    d = normalize(voxel_to_light)
    o = p - 1000.0 * d
    
    # Intersect ray (o, d) with box
    box_hit = ray_box_intersect(o, d, box_min, box_max)
    if box_hit.y < box_hit.x:
        return 1.0
        
    integrated = 0.0
    delta = min(voxel_size.x, min(voxel_size.y, voxel_size.z)) / 1.0 # samplesPerVoxel = 1.0
    t = box_hit.x
    while t < box_hit.y:
        sample_pos = o + t * d
        val = sample_volume_opacity(sample_pos, ...) # Simplified
        integrated += val / 4.0
        if integrated > 0.95:
            break
        t += delta
    return clamp(integrated, 0.0, 1.0)
```

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$ per pixel, where $N$ is the number of ray march steps and $M$ is the number of shadow ray samples (typically 1 in this implementation, but the structure allows for more). The shadow calculation adds a significant constant factor.
*   **Memory Complexity**: $O(V)$ for the volume texture, plus $O(C)$ for the transfer function control points.
*   **Performance**: Designed for interactivity in web browsers. Performance depends heavily on `dt_scale` and volume resolution. Early termination at $\alpha=0.95$ helps performance in opaque regions.
*   **Strengths**: Runs on standard web hardware without plugins. Supports arbitrary transfer functions. Provides basic shading (diffuse/specular/shadow) for better depth perception.
*   **Limitations**: Linear search for transfer functions is slower than texture-based lookup. Shadow calculation is expensive and simplified (single sample per voxel direction). No global illumination or caustics.
*   **When to use**: Interactive visualization of scalar fields in web applications where basic shading improves understanding.
*   **When not to use**: High-fidelity scientific visualization requiring accurate light transport, or very large volumes where browser memory limits are exceeded.

## Background
This method is based on the open-source project "webgl-volumerendering" by termijn, which demonstrates interactive voxel volume rendering using WebGL 2.0 and path tracing techniques in a browser environment [S1, unofficial]. It builds upon the principles of direct volume ray casting [19-direct-volume-ray-casting.md] but adapts them for the fragment shader execution model of WebGL, incorporating shading models typically found in path tracers.

## Sources
[S1] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
