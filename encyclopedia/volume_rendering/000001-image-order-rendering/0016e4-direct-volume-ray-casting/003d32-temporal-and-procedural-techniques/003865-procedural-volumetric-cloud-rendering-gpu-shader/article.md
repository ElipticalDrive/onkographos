# Procedural Volumetric Cloud Rendering (GPU Shader)

Also called: CloudRendererProcedural, Procedural GPU Implementation

## Overview
This method renders volumetric clouds by evaluating procedural noise functions directly within a GPU fragment shader during a direct volume ray casting pass [S1, unofficial]. It solves the problem of generating high-resolution, animated cloud geometry without the memory overhead of pre-computed 3D textures or the latency of CPU-side simulation [S1, unofficial]. It distinguishes itself from texture-based volume rendering by computing density analytically per sample, allowing for infinite resolution and seamless animation, while differing from CPU-side marching by leveraging parallel GPU execution for real-time performance [S1, unofficial][S2, unofficial].

## How it works
The method operates as a post-processing or camera-attached component that intercepts the rendered scene, casts rays through a defined volumetric bounding box, and composites the result with the original scene color [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `_MainTex`: The original scene color buffer (sampler2D) [S1, unofficial].
    *   `_CameraWorldPosition`: The world-space position of the camera [S1, unofficial].
    *   `input.rayDirection`: The view ray direction for the current pixel, derived from the camera projection [S1, unofficial].
    *   `_CloudVolumeMinBound`, `_CloudVolumeMaxBound`: Axis-aligned bounding box (AABB) defining the cloud volume in world space [S1, unofficial].
    *   `_SunDirection`: Normalized vector indicating the direction of the sun [S1, unofficial].
    *   `_SunColour`: RGB color of the sun [S1, unofficial].
    *   `_NoiseTex`: A 3D texture containing procedural noise data (typically Worley or FBM) [S1, unofficial].
    *   Parameters: `_AbsorptionCoefficient`, `_ScatterFactor`, `Density Threshold`, `Cloud Scale`, `Octave Count`, `Wind Speed` [S1, unofficial].
*   **Outputs**:
    *   Final pixel color: A `float4` representing the composited cloud color and alpha [S1, unofficial].

### Data Structures
The implementation relies on standard vector and scalar types. No complex custom structures are defined in the source; it uses `float3` for positions/vectors, `float2` for intersection distances, and `float4` for colors [S1, unofficial].

### Algorithm Steps

1.  **Retrieve Background**: Sample the original scene color from `_MainTex` using the current UV coordinates: `originalColour = tex2D(_MainTex, input.uv)` [S1, unofficial].
2.  **Initialize Ray**:
    *   Set ray origin: `origin = _CameraWorldPosition` [S1, unofficial].
    *   Normalize ray direction: `direction = normalize(input.rayDirection)` [S1, unofficial]. *Pitfall*: Failure to normalize causes the ray to step past the volume instantly due to large unnormalized values relative to the far clip plane [S1, unofficial].
3.  **Compute Intersection**: Calculate the entry and exit points of the ray with the cloud volume box using the slab method [S1, unofficial]:
    *   Compute distance to each box face along the ray axes.
    *   `entries = min(entryDist, exitDist)` for each axis.
    *   `exits = max(entryDist, exitDist)` for each axis.
    *   `entry = max(max(entries.x, entries.y), entries.z)`
    *   `exit = min(min(exits.x, exits.y), exits.z)`
    *   If `entry < exit`, the ray intersects the volume [S1, unofficial].
4.  **March Density**: If intersecting, call `MarchDensity(entry, exit, origin, direction, ...)` [S1, unofficial].
    *   Initialize `brightness = 0`, `densityTotal = 0`, `distanceTravelled = entry` [S1, unofficial].
    *   Set `stepSize` (source implies a fixed or derived step size, typically small enough to capture gradients) [S1, unofficial].
    *   Loop up to `stepLimit` (e.g., 64 iterations) [S1, unofficial]:
        *   If `distanceTravelled > exit`, break [S1, unofficial].
        *   Update position: `rayPosition = rayOrigin + rayDirection * distanceTravelled` [S1, unofficial].
        *   Compute world-scaled coordinates: `worldScaledRay = rayPosition * _CloudScale` [S1, unofficial].
        *   **Procedural Density Evaluation**: Compute Worley noise density using Fractal Brownian Motion (FBM) [S1, unofficial]:
            *   Initialize `worleyDensity = 0`, `amplitude = 1.0`, `frequency = 1.0`, `amplitudeTotal = 0` [S1, unofficial].
            *   Loop `octave` from 0 to `_OctaveCount - 1`:
                *   Sample noise: `sample = tex3D(_NoiseTex, worldScaledRay * frequency).r` [S1, unofficial].
                *   Accumulate: `worleyDensity += sample * amplitude` [S1, unofficial].
                *   Update amplitude/frequency: `amplitude *= 0.5`, `frequency *= 2.0`, `amplitudeTotal += amplitude` [S1, unofficial].
            *   Normalize: `worleyDensity = worleyDensity / amplitudeTotal` (if `amplitudeTotal > 0`) [S1, unofficial].
            *   Apply density threshold: If `worleyDensity < Density Threshold`, treat density as 0 [S1, unofficial].
        *   **Light Transmittance**: Calculate sunlight transmittance to the current sample point using `MarchLight(rayPosition, boxMin, boxMax)` [S1, unofficial]:
            *   Initialize `lightDensity = 0` [S1, unofficial].
            *   March from `rayPosition` in direction `_SunDirection` for `stepLimit` (e.g., 200) steps of size 20.0 [S1, unofficial].
            *   For each step, if inside the box, accumulate noise samples scaled by step size into `lightDensity` [S1, unofficial].
            *   Compute transmittance: `sunTransmittance = exp(-lightDensity * _AbsorptionCoefficient)` [S1, unofficial].
        *   **Camera Transmittance**: Calculate transmittance from the sample to the camera (viewer) using accumulated density so far: `cameraTransmittance = exp(-densityTotal * _AbsorptionCoefficient)` [S1, unofficial].
        *   **Scattering**: Compute Henyey-Greenstein scattering factor `scatter` using `rayDirection` and `_SunDirection` [S1, unofficial]:
            *   `cosAngle = dot(normalize(rayDirection), normalize(_SunDirection))` [S1, unofficial].
            *   `scatterFactor2 = _ScatterFactor * _ScatterFactor` [S1, unofficial].
            *   `numerator = 1.0 - scatterFactor2` [S1, unofficial].
            *   `denominator = pow(1.0 + scatterFactor2 - 2.0 * _ScatterFactor * cosAngle, 1.5)` [S1, unofficial].
            *   `scatter = numerator / (4.0 * PI * denominator)` [S1, unofficial].
        *   **Accumulate Brightness**:
            *   `brightness = brightness + density * sunTransmittance * cameraTransmittance * scatter * stepSize * 0.5` [S1, unofficial].
        *   **Accumulate Density**:
            *   `densityTotal = densityTotal + density * stepSize` [S1, unofficial].
        *   **Increment Distance**:
            *   `distanceTravelled = distanceTravelled + stepSize` [S1, unofficial].
5.  **Final Color Calculation**:
    *   Compute cloud transmittance: `cloudTransmittance = exp(-densityTotal * _AbsorptionCoefficient)` [S1, unofficial].
    *   Compute cloud color: `cloudColour = brightness * _SunColour.rgb` [S1, unofficial].
    *   Return alpha: `alpha = 1 - cloudTransmittance` [S1, unofficial].
6.  **Composite**: Blend cloud color with background: `return lerp(originalColour, cloudColour, cloudColour.a)` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ProceduralCloudRender(input_uv, input_ray_direction, camera_world_pos, 
                          cloud_volume_min, cloud_volume_max, sun_direction, sun_colour,
                          noise_tex, absorption_coefficient, scatter_factor, 
                          density_threshold, cloud_scale, octave_count):
    # 1. Retrieve Background
    original_colour = sample_texture_2d(noise_tex, input_uv) # Placeholder for _MainTex
    
    # 2. Initialize Ray
    origin = camera_world_pos
    direction = normalize(input_ray_direction)
    
    # 3. Compute Intersection (Slab Method)
    entry_dist = (cloud_volume_min - origin) / direction
    exit_dist = (cloud_volume_max - origin) / direction
    
    entries = min(entry_dist, exit_dist)
    exits = max(entry_dist, exit_dist)
    
    entry = max(max(entries.x, entries.y), entries.z)
    exit = min(min(exits.x, exits.y), exits.z)
    
    if entry >= exit:
        return original_colour
    
    # 4. March Density
    brightness = 0.0
    density_total = 0.0
    distance_travelled = entry
    step_size = 1.0 # Assumed constant step size for simplicity; source implies fixed loop
    
    # Limit iterations to prevent infinite loops
    step_limit = 64 
    
    for i in range(step_limit):
        if distance_travelled > exit:
            break
            
        ray_position = origin + direction * distance_travelled
        world_scaled_ray = ray_position * cloud_scale
        
        # Procedural Density Evaluation (Worley FBM)
        worley_density = 0.0
        amplitude = 1.0
        frequency = 1.0
        amplitude_total = 0.0
        
        for octave in range(octave_count):
            sample = sample_texture_3d(noise_tex, world_scaled_ray * frequency).r
            worley_density += sample * amplitude
            amplitude_total += amplitude
            amplitude *= 0.5
            frequency *= 2.0
            
        if amplitude_total > 0:
            worley_density /= amplitude_total
        else:
            worley_density = 0.0
            
        # Apply Density Threshold
        if worley_density < density_threshold:
            density = 0.0
        else:
            density = worley_density
            
        # Light Transmittance (March towards sun)
        sun_transmittance = march_light(ray_position, cloud_volume_min, cloud_volume_max, 
                                        sun_direction, noise_tex, cloud_scale, absorption_coefficient)
        
        # Camera Transmittance
        camera_transmittance = exp(-density_total * absorption_coefficient)
        
        # Scattering (Henyey-Greenstein)
        cos_angle = dot(normalize(direction), normalize(sun_direction))
        scatter_factor_sq = scatter_factor * scatter_factor
        numerator = 1.0 - scatter_factor_sq
        denominator = pow(1.0 + scatter_factor_sq - 2.0 * scatter_factor * cos_angle, 1.5)
        scatter = numerator / (4.0 * 3.14159 * denominator)
        
        # Accumulate Brightness
        brightness += density * sun_transmittance * camera_transmittance * scatter * step_size * 0.5
        
        # Accumulate Density
        density_total += density * step_size
        
        # Increment Distance
        distance_travelled += step_size
        
    # 5. Final Color Calculation
    cloud_transmittance = exp(-density_total * absorption_coefficient)
    cloud_colour = brightness * sun_colour.rgb
    alpha = 1.0 - cloud_transmittance
    
    # 6. Composite
    return lerp(original_colour, cloud_colour, alpha)

def march_light(ray_position, box_min, box_max, sun_direction, noise_tex, cloud_scale, absorption_coefficient):
    step_size_light = 20.0
    step_limit_light = 200
    light_density = 0.0
    
    current_pos = ray_position
    for i in range(step_limit_light):
        current_pos += sun_direction * step_size_light
        
        # Check if inside box
        uvw = (current_pos - box_min) / (box_max - box_min)
        if 0 <= uvw.x <= 1 and 0 <= uvw.y <= 1 and 0 <= uvw.z <= 1:
            world_scaled = current_pos * cloud_scale
            light_density += sample_texture_3d(noise_tex, world_scaled).r * step_size_light
            
    return exp(-light_density * absorption_coefficient)
```

### Parameters
*   `_AbsorptionCoefficient`: Controls light absorption via Beer-Lambert transmittance. Higher values result in darker clouds [S1, unofficial].
*   `_ScatterFactor`: Controls forward light scattering via Henyey-Greenstein scattering. Higher values result in stronger silver linings when the sun is behind clouds [S1, unofficial].
*   `Density Threshold`: The minimum noise value to contribute to cloud density. Higher values create more gaps between clouds [S1, unofficial].
*   `Cloud Scale`: Controls the size of cloud features in world space. Higher values result in smaller clouds [S1, unofficial].
*   `Octave Count`: Number of Fractal Brownian Motion (FBM) layers. Higher values result in more detailed cloud edges [S1, unofficial].
*   `Wind Speed`: Controls cloud animation speed by offsetting noise sample coordinates (implied by "Wind Speed" parameter and noise animation context) [S1, unofficial].

### Edge Cases and Pitfalls
*   **Unnormalized Ray Directions**: If `input.rayDirection` is not normalized, the ray may step past the entire volume instantly because the distance read relative to the far clip plane is too large, preventing density gradients from being shown [S1, unofficial].
*   **Division by Zero**: In FBM normalization, if `amplitudeTotal` is 0, density is set to 0 to avoid division by zero [S1, unofficial].
*   **Loop Limits**: The source refactored a `while` loop to a `for` loop with a defined `stepLimit` (64 for camera ray, 200 for light ray) to prevent shader errors caused by variable iteration counts [S1, unofficial].

## Subtopics

### CPU-side Ray Marching for Volumetric Clouds
This variant performs ray marching on the CPU rather than the GPU [S2, unofficial]. It renders volumetric clouds by marching rays through a defined 3D box using noise for density and opacity parameters [S2, unofficial]. It is called via `renderer.RenderVolumetricClouds(box_min, box_max, density, noise_scale, octaves, opacity)` each frame [S2, unofficial]. Parameters include `cloud_box_min` (default -100), `cloud_box_max` (default 100), `cloud_density`, `cloud_noise_scale`, `cloud_octaves`, and `cloud_opacity` [S2, unofficial]. A major pitfall is the performance bottleneck due to CPU-side execution, which is mitigated by using 4x4 pixel striding, though this reduces quality [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of samples per ray and the number of rays (pixels). The complexity is $O(N_{pixels} \times N_{steps})$, where $N_{steps}$ is determined by `stepLimit` and `stepSize` [S1, unofficial].
*   **Memory Complexity**: Low. It does not require storing a 3D volume texture for density if procedural noise is computed on the fly, though the source uses a `_NoiseTex` 3D texture, implying memory usage proportional to the noise texture size [S1, unofficial].
*   **Measured Performance**: The GPU implementation is designed for real-time performance, unlike the CPU variant which suffers from bottlenecks [S1, unofficial][S2, unofficial].
*   **Strengths**: Infinite resolution, no pre-computation overhead, seamless animation via noise offsets, and efficient parallel execution on GPU [S1, unofficial].
*   **Limitations**: Computationally expensive per pixel due to procedural noise evaluation and multiple ray marches (one for density, one for light) [S1, unofficial].
*   **When to Use**: Real-time rendering of dynamic clouds where memory bandwidth for large 3D textures is a constraint or where animation flexibility is required [S1, unofficial].
*   **When Not to Use**: When CPU-only rendering is available and performance is not critical, or when static, pre-baked lighting is preferred over dynamic sun interaction [S2, unofficial].

## Background
This method is implemented in Unity as the `CloudRendererProcedural` component, attached to the main camera [S1, unofficial]. It builds on the principles of direct volume ray casting and procedural noise generation. The Henyey-Greenstein scattering function and Beer-Lambert transmittance are standard techniques in physically based rendering, often cited in industry whitepapers such as those from EA Frostbite [S1, unofficial].

## Sources
[S1] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S2] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
