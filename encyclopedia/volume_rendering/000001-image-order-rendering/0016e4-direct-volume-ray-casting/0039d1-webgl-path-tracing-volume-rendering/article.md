# WebGL Path Tracing Volume Rendering
Also called: Interactive Voxel Volume Rendering via WebGL 2.0

## Overview
WebGL Path Tracing Volume Rendering is an interactive voxel-based volume rendering technique implemented in modern web browsers using WebGL 2.0 [S1, unofficial]. It renders 3D scalar fields by casting rays from the camera through the volume, accumulating color and opacity along the ray path, and applying lighting and shadowing effects [S1, unofficial]. It distinguishes itself from standard direct volume ray casting by integrating a path-tracing-like lighting model (diffuse, specular, and soft shadows) directly into the fragment shader, enabling real-time interactive performance on client-side hardware [S1, unofficial].

## How it works
The method operates as a fragment shader program that processes each pixel of the viewport. It builds upon the standard direct volume ray casting loop defined in [Direct volume ray casting](19-direct-volume-ray-casting.md), specifically utilizing the ray generation, sampling, classification, and compositing stages [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   `volume`: A `highp sampler3D` containing the scalar field data.
*   `volume_dims`: An `ivec3` specifying the dimensions of the volume texture.
*   `vray_dir`: A `vec3` representing the ray direction in world space.
*   `transformed_eye`: A `vec3` representing the camera origin in world space.
*   `lightPosition`: A `vec3` specifying the position of the light source.
*   `windowLevel` and `windowWidth`: Floats defining the windowing (level/width) for the transfer function.
*   `rescaleSlope` and `rescaleIntercept`: Floats for linear transformation of raw volume values.
*   `dt_scale`: A float scaling factor for the ray marching step size.
*   `opacityMapPositions` and `opacityMapSamples`: `highp sampler2D` textures storing the control points and values for the opacity transfer function.
*   `colorMapPositions` and `colorMapSamples`: `highp sampler2D` textures storing the control points and values for the color transfer function.
*   `numberOfOpacitySamples` and `numberOfColorSamples`: Integers specifying the number of control points.

**Output:**
*   `color`: A `vec4` representing the final RGBA color of the pixel.

### Data Structures
The transfer functions are stored in 2D textures where the x-coordinate represents the position (scalar value) and the y-coordinate represents the sample value (opacity or color). The positions are not necessarily uniform, requiring a linear search to find the correct interval [S1, unofficial].

### Algorithm Steps

1.  **Ray-Box Intersection**:
    Compute the intersection of the ray $r(t) = \mathbf{o} + t\mathbf{d}$ with the volume bounding box defined by `boxMin` and `boxMax`.
    *   Compute `invDir = 1.0 / dir`.
    *   Compute `tmin_tmp = (boxMin - orig) * invDir` and `tmax_tmp = (boxMax - orig) * invDir`.
    *   Compute `tmin = min(tmin_tmp, tmax_tmp)` and `tmax = max(tmin_tmp, tmax_tmp)`.
    *   Compute `t0 = max(tmin.x, max(tmin.y, tmin.z))` and `t1 = min(tmax.x, min(tmax.y, tmax.z))`.
    *   If `t0 > t1`, the ray does not intersect the volume; discard the fragment [S1, unofficial].

2.  **Step Size Calculation**:
    Determine the step size `dt` based on the volume dimensions and ray direction to ensure consistent sampling density relative to voxel size.
    *   Compute `dt_vec = 1.0 / (vec3(volume_dims) * abs(ray_dir))`.
    *   Compute `dt = dt_scale * min(dt_vec.x, min(dt_vec.y, dt_vec.z))` [S1, unofficial].

3.  **Ray Origin Offset**:
    Apply a random offset to the ray origin to reduce aliasing artifacts.
    *   Generate a pseudo-random number using the Wang hash function based on `gl_FragCoord`.
    *   Compute initial position `p = transformed_eye + (t0 + offset * dt) * ray_dir` [S1, unofficial].

4.  **Ray Marching Loop**:
    Iterate from `t0` to `t1` with step `dt`. Initialize accumulated color `voxelColor = vec4(0.0)`.
    *   **Sampling**: Sample the volume value `val` at position `p` using trilinear interpolation from `volume`.
    *   **Rescaling**: Apply linear transformation: `rescaled_value = rescaleSlope * val + rescaleIntercept`.
    *   **Windowing**: Normalize the value for the transfer function: `normalized_val = (rescaled_value - (windowLevel - windowWidth / 2.0)) / windowWidth`.
    *   **Classification**:
        *   **Opacity**: Find the interval in `opacityMapPositions` containing `normalized_val`. Interpolate between the corresponding `opacityMapSamples` using linear interpolation. Clamp the result to $[0, 1]$.
        *   **Color**: Similarly, find the interval in `colorMapPositions` and interpolate `colorMapSamples` to get the base color $C_i$.
    *   **Beer-Lambert Correction**: Adjust opacity for the step size. Let $\alpha_i$ be the interpolated opacity. Compute corrected alpha: $\alpha'_i = 1.0 - (1.0 - \alpha_i)^{dt\_scale}$.
    *   **Compositing**:
        *   Compute `val_color = vec4(opacity * color, opacity)`. Note: The source implementation uses `val_color.a` as the corrected alpha $\alpha'_i$ and `val_color.rgb` as the contribution.
        *   Update accumulated color: `voxelColor.rgb += (1.0 - voxelColor.a) * val_color.a * val_color.rgb`.
        *   Update accumulated alpha: `voxelColor.a += (1.0 - voxelColor.a) * val_color.a`.
    *   **Early Termination**: If `voxelColor.a >= 0.95`, break the loop [S1, unofficial].
    *   **Advance**: `p += ray_dir * dt` [S1, unofficial].

5.  **Lighting Calculation**:
    After the loop, compute lighting at the final position `p` (where the ray exited or terminated).
    *   **Normal Estimation**: Compute the gradient of the opacity field at `p` using central differences.
        *   Compute `voxelSize = 1.0 / vec3(volume_dims)`.
        *   Compute `gradient.x = (sampleVolumeOpacity(p + vec3(voxelSize.x, 0, 0)) - sampleVolumeOpacity(p - vec3(voxelSize.x, 0, 0))) / (2.0 * voxelSize.x)`, similarly for y and z.
        *   Normalize the gradient to get `voxelNormal` [S1, unofficial].
    *   **Diffuse**: Compute `lightDirection = normalize(lightPosition - p)`. Compute `intensity = max(dot(voxelNormal, lightDirection), 0.0)`.
    *   **Specular**: If `intensity > 0.0`, compute half vector `h = normalize(lightDirection + normalize(-transformed_eye))`. Compute `intSpec = max(dot(h, voxelNormal), 0.0)`. Compute `spec = specular * pow(intSpec, shininess)` with `shininess = 8.0` [S1, unofficial].
    *   **Shadow**: If `voxelColor.a >= 0.20`, compute `shadow = clamp(softShadow(p) * 2.0 + 0.1, 0.0, 1.0)`. Else `shadow = 1.0`.
    *   **Final Light**: `lightColor = shadow * max(intensity * diffuse + spec, ambient)` with `diffuse=0.6`, `ambient=0.4`, `specular=0.5` [S1, unofficial].
    *   Apply `lightColor` to the accumulated `voxelColor` [S1, unofficial].

6.  **Output**:
    Convert the linear color to sRGB and output as the fragment color [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def webgl_path_tracing_volume_rendering(vray_dir, transformed_eye, volume, volume_dims, light_position, window_level, window_width, rescale_slope, rescale_intercept, dt_scale, opacity_map_positions, opacity_map_samples, color_map_positions, color_map_samples, number_of_opacity_samples, number_of_color_samples):
    # 1. Ray-Box Intersection
    inv_dir = 1.0 / vray_dir
    tmin_tmp = (box_min - transformed_eye) * inv_dir
    tmax_tmp = (box_max - transformed_eye) * inv_dir
    tmin = min(tmin_tmp, tmax_tmp)
    tmax = max(tmin_tmp, tmax_tmp)
    t0 = max(tmin.x, max(tmin.y, tmin.z))
    t1 = min(tmax.x, min(tmax.y, tmax.z))
    
    if t0 > t1:
        return vec4(0.0, 0.0, 0.0, 0.0) # Discard fragment

    # 2. Step Size Calculation
    dt_vec = 1.0 / (vec3(volume_dims) * abs(vray_dir))
    dt = dt_scale * min(dt_vec.x, min(dt_vec.y, dt_vec.z))

    # 3. Ray Origin Offset
    seed = int(gl_frag_coord.x + 1024.0 * gl_frag_coord.y)
    offset = wang_hash(seed)
    p = transformed_eye + (t0 + offset * dt) * vray_dir

    # 4. Ray Marching Loop
    voxel_color = vec4(0.0, 0.0, 0.0, 0.0)
    t = t0
    while t < t1:
        # Sampling
        val = sample_volume(volume, p)
        
        # Rescaling and Windowing
        rescaled_value = rescale_slope * val + rescale_intercept
        normalized_val = (rescaled_value - (window_level - window_width / 2.0)) / window_width
        
        # Classification: Opacity
        alpha_i = interpolate_transfer_function(normalized_val, opacity_map_positions, opacity_map_samples, number_of_opacity_samples)
        alpha_i = clamp(alpha_i, 0.0, 1.0)
        
        # Classification: Color
        color_i = interpolate_transfer_function_color(normalized_val, color_map_positions, color_map_samples, number_of_color_samples)
        
        # Beer-Lambert Correction
        alpha_prime = 1.0 - pow(1.0 - alpha_i, dt_scale)
        
        # Compositing
        voxel_color.rgb += (1.0 - voxel_color.a) * alpha_prime * color_i
        voxel_color.a += (1.0 - voxel_color.a) * alpha_prime
        
        if voxel_color.a >= 0.95:
            break
            
        t += dt
        p += vray_dir * dt

    # 5. Lighting Calculation
    voxel_size = 1.0 / vec3(volume_dims)
    gradient = compute_gradient_opacity(p, volume, opacity_map_positions, opacity_map_samples, number_of_opacity_samples, window_level, window_width, rescale_slope, rescale_intercept, voxel_size)
    voxel_normal = normalize(gradient)
    
    light_direction = normalize(light_position - p)
    intensity = max(dot(voxel_normal, light_direction), 0.0)
    
    specular = 0.5
    shininess = 8.0
    spec = 0.0
    if intensity > 0.0:
        h = normalize(light_direction + normalize(-transformed_eye))
        int_spec = max(dot(h, voxel_normal), 0.0)
        spec = specular * pow(int_spec, shininess)
        
    shadow = 1.0
    if voxel_color.a >= 0.20:
        shadow = clamp(soft_shadow(p, light_position, volume, opacity_map_positions, opacity_map_samples, number_of_opacity_samples, window_level, window_width, rescale_slope, rescale_intercept, voxel_size) * 2.0 + 0.1, 0.0, 1.0)
        
    diffuse = 0.6
    ambient = 0.4
    light_color = shadow * max(intensity * diffuse + spec, ambient)
    
    voxel_color.rgb *= light_color

    # 6. SRGB Conversion
    return linear_to_srgb(voxel_color)

def wang_hash(seed):
    seed = (seed ^ 61) ^ (seed >> 16)
    seed *= 9
    seed = seed ^ (seed >> 4)
    seed *= 0x27d4eb2d
    seed = seed ^ (seed >> 15)
    return float(seed % 2147483647) / float(2147483647)

def interpolate_transfer_function(value, positions, samples, num_samples):
    # Linear search for interval
    for i in range(1, num_samples):
        if positions[i] >= value:
            # Linear interpolation
            t = (value - positions[i-1]) / (positions[i] - positions[i-1])
            return samples[i-1] + t * (samples[i] - samples[i-1])
    return samples[num_samples-1]

def compute_gradient_opacity(p, volume, opacity_map_positions, opacity_map_samples, num_opacity_samples, window_level, window_width, rescale_slope, rescale_intercept, voxel_size):
    # Central differences of opacity
    def sample_opacity(pos):
        val = sample_volume(volume, pos)
        rescaled = rescale_slope * val + rescale_intercept
        normalized = (rescaled - (window_level - window_width / 2.0)) / window_width
        return interpolate_transfer_function(normalized, opacity_map_positions, opacity_map_samples, num_opacity_samples)
        
    grad_x = (sample_opacity(p + vec3(voxel_size.x, 0, 0)) - sample_opacity(p - vec3(voxel_size.x, 0, 0))) / (2.0 * voxel_size.x)
    grad_y = (sample_opacity(p + vec3(0, voxel_size.y, 0)) - sample_opacity(p - vec3(0, voxel_size.y, 0))) / (2.0 * voxel_size.y)
    grad_z = (sample_opacity(p + vec3(0, 0, voxel_size.z)) - sample_opacity(p - vec3(0, 0, voxel_size.z))) / (2.0 * voxel_size.z)
    return vec3(grad_x, grad_y, grad_z)

def soft_shadow(p, light_position, volume, opacity_map_positions, opacity_map_samples, num_opacity_samples, window_level, window_width, rescale_slope, rescale_intercept, voxel_size):
    # Trace light from p to light
    voxel_to_light = light_position - p
    d = normalize(voxel_to_light)
    o = p - 1000.0 * d # Start ray far behind p
    
    # Intersect with box (simplified, assuming box is large enough or handled by logic)
    # In source, it intersects ray with box to get start and end
    # For implementation, we assume the ray starts at p and goes to light
    # The source code says: "Sample opacity along the ray from start to box.y"
    # We will approximate the shadow ray traversal from p to light
    
    delta = min(voxel_size.x, min(voxel_size.y, voxel_size.z)) / 1.0 # samplesPerVoxel = 1.0
    integrated = 0.0
    current_pos = p
    direction_to_light = d
    
    # Traverse from p to light
    # Note: Source uses a specific box intersection logic for the shadow ray.
    # Here we simulate the accumulation along the path to the light.
    # The source breaks if integrated > 0.95.
    
    # We need the distance to the light to bound the loop
    dist_to_light = length(light_position - p)
    
    t = 0.0
    while t < dist_to_light:
        val = sample_volume(volume, current_pos)
        rescaled = rescale_slope * val + rescale_intercept
        normalized = (rescaled - (window_level - window_width / 2.0)) / window_width
        alpha = interpolate_transfer_function(normalized, opacity_map_positions, opacity_map_samples, num_opacity_samples)
        
        # Accumulate opacity
        integrated += alpha / 1.0 / 4.0 # Source divides by samplesPerVoxel (1.0) and 4.0
        
        if integrated > 0.95:
            break
            
        t += delta
        current_pos += direction_to_light * delta
        
    return clamp(integrated, 0.0, 1.0)

def linear_to_srgb(x):
    if x <= 0.0031308:
        return 12.92 * x
    else:
        return 1.055 * pow(x, 1.0 / 2.4) - 0.055
```

### Parameters
*   `dt_scale`: Scaling factor for the step size. Affects the number of samples and the accuracy of the Beer-Lambert approximation. Typical values are not specified in the source, but are adjusted via a "Sampling rate" slider (0.25x to 8x) [S1, unofficial].
*   `rescaleSlope` and `rescaleIntercept`: Linear transformation parameters for the raw volume data. Typical values depend on the specific volume dataset [S1, unofficial].
*   `windowLevel` and `windowWidth`: Define the range of scalar values mapped to the transfer function. Typical values depend on the dataset's Hounsfield units or intensity range [S1, unofficial].

### Edge Cases and Pitfalls
*   **Ray-Box Intersection Failure**: If `t0 > t1`, the ray misses the volume. The fragment is discarded [S1, unofficial].
*   **Early Termination**: The ray marching loop breaks if `voxelColor.a >= 0.95`. This may lead to slightly inaccurate transparency if the volume is highly transparent but the threshold is too low [S1, unofficial].
*   **Shadow Ray Early Termination**: The shadow ray loop breaks if `integrated > 0.95`. This approximates the shadow and may miss subtle lighting variations [S1, unofficial].
*   **Transfer Function Lookup**: The opacity and color maps use a linear search through control points rather than texture sampling. This allows for arbitrary non-uniform control point positions but is computationally more expensive than texture-based lookup [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: The complexity is proportional to the number of samples along the ray, which depends on the volume dimensions, ray direction, and `dt_scale`. The shadow ray adds a secondary traversal, increasing the cost per pixel.
*   **Memory Complexity**: The method requires storing the volume data in a 3D texture and the transfer functions in 2D textures. The memory usage is dominated by the volume data size.
*   **Performance**: Designed for interactive performance in web browsers. Performance scales with the resolution of the viewport and the sampling rate. The use of WebGL 2.0 allows for hardware acceleration on supported GPUs [S1, unofficial].
*   **Strengths**: Interactive, runs in standard web browsers, supports arbitrary transfer functions, includes lighting and shadows.
*   **Limitations**: Limited to scalar volumes, relies on fragment shader performance, shadow calculation is approximate and can be slow for complex scenes [S1, unofficial].

## Background
The method is implemented in the repository `termijn/webgl-volumerendering` [S1, unofficial]. It demonstrates the capabilities of WebGL 2.0 for real-time volume rendering with advanced lighting effects. The implementation uses standard graphics techniques such as ray-box intersection, trilinear interpolation, and front-to-back compositing, extended with a path-tracing-like lighting model [S1, unofficial].

## Sources
[S1] termijn/webgl-volumerendering, https://github.com/termijn/webgl-volumerendering, fetched 2026-10-27 (unofficial)
