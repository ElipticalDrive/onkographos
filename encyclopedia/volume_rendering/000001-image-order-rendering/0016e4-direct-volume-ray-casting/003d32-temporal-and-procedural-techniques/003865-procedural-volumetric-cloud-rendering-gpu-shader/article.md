# Procedural Volumetric Cloud Rendering (GPU Shader)

Also called: CloudRendererProcedural, Procedural GPU Implementation

## Overview
This method renders volumetric clouds by evaluating density and lighting on-the-fly within a GPU fragment shader, eliminating the need for pre-computed 3D texture lookups or CPU-side ray marching. It solves the performance bottleneck of traditional volume rendering by using procedural noise (specifically Fractal Brownian Motion) to generate cloud structure and a simplified Henyey-Greenstein phase function for scattering [S1, unofficial]. Unlike sibling methods that rely on fixed volumetric textures or CPU execution, this approach allows for dynamic, infinite-resolution cloud features and real-time animation via wind parameters, making it suitable for real-time engines like Unity [S1, unofficial].

## How it works
The method operates as a post-process or camera-attached component that intercepts the scene color and overlays procedurally generated clouds. It builds on the ray casting loop structure defined in [Direct volume ray casting](19-direct-volume-ray-casting.md) but replaces texture sampling with procedural noise evaluation and uses a simplified lighting model.

### Inputs and Outputs
*   **Inputs**:
    *   `originalColour`: The background scene color sampled from the main render texture at the current pixel UV [S1, unofficial].
    *   `rayDirection`: The direction vector from the camera to the pixel, derived from screen space [S1, unofficial].
    *   `_CameraWorldPosition`: The world-space position of the camera [S1, unofficial].
    *   `_CloudVolumeMinBound`, `_CloudVolumeMaxBound`: Axis-aligned bounding box (AABB) defining the cloud volume in world space [S1, unofficial].
    *   `_SunDirection`: Normalized direction vector to the sun [S1, unofficial].
    *   `_SunColour`: RGB color of the sun [S1, unofficial].
    *   Parameters: `_AbsorptionCoefficient`, `_ScatterFactor`, `Density Threshold`, `Cloud Scale`, `Octave Count`, `Wind Speed` [S1, unofficial].
*   **Outputs**:
    *   `finalColour`: The composited color of the cloud and background [S1, unofficial].

### Data Structures
The implementation relies on standard vector types (`float3`, `float4`) and scalar floats. No complex data structures are required on the GPU side; all state is passed via shader constants or uniforms [S1, unofficial].

### Algorithm Steps

1.  **Retrieve Background**: Sample the original scene color at the current pixel coordinates:
    $$ C_{bg} = \text{tex2D}(\text{_MainTex}, uv) $$
    [S1, unofficial]

2.  **Initialize Ray**:
    *   Set ray origin $\mathbf{o}$ to `_CameraWorldPosition` [S1, unofficial].
    *   Normalize the ray direction $\mathbf{d}$ to prevent stepping errors due to large unnormalized values:
        $$ \mathbf{d} = \text{normalize}(\text{input.rayDirection}) $$
        [S1, unofficial]

3.  **Ray-Box Intersection**: Compute the entry and exit distances of the ray with the cloud volume AABB using the slab method [S1, unofficial]:
    *   Calculate intersection distances for each axis $i \in \{x, y, z\}$:
        $$ t_{i,entry} = \frac{\text{MinBound}_i - o_i}{d_i}, \quad t_{i,exit} = \frac{\text{MaxBound}_i - o_i}{d_i} $$
    *   Determine global entry and exit:
        $$ t_{entry} = \max(t_{x,entry}, t_{y,entry}, t_{z,entry}) $$
        $$ t_{exit} = \min(t_{x,exit}, t_{y,exit}, t_{z,exit}) $$
    *   If $t_{entry} > t_{exit}$, the ray misses the volume; return $C_{bg}$ [S1, unofficial].

4.  **March Density and Accumulate Brightness**:
    *   Initialize `brightness` = 0, `densityTotal` = 0, `distanceTravelled` = 0 [S1, unofficial].
    *   Set step size (e.g., 20.0 units) and step limit (e.g., 64 iterations) [S1, unofficial].
    *   Start ray position $\mathbf{p} = \mathbf{o} + \mathbf{d} \times t_{entry}$ [S1, unofficial].
    *   Loop for `i` from 0 to `stepLimit`:
        *   If `distanceTravelled` > $t_{exit}$, break [S1, unofficial].
        *   Advance position: $\mathbf{p} = \mathbf{p} + \mathbf{d} \times \text{stepSize}$ [S1, unofficial].
        *   Calculate world-scaled coordinates for noise sampling:
            $$ \mathbf{p}_{scaled} = \mathbf{p} \times \text{_CloudScale} $$
            [S1, unofficial]
        *   **Procedural Density (FBM)**:
            *   Initialize `worleyDensity` = 0, `amplitude` = 1.0, `frequency` = 1.0, `amplitudeTotal` = 0 [S1, unofficial].
            *   Loop for `octave` from 0 to `_OctaveCount` - 1:
                *   Sample noise (e.g., Worley or similar) at $\mathbf{p}_{scaled} \times \text{frequency}$ [S1, unofficial].
                *   Add to density: `worleyDensity` += `noiseValue` $\times$ `amplitude` [S1, unofficial].
                *   Accumulate amplitude: `amplitudeTotal` += `amplitude` [S1, unofficial].
                *   Scale amplitude: `amplitude` $\times= 0.5$ [S1, unofficial].
                *   Scale frequency: `frequency` $\times= 2.0$ [S1, unofficial].
            *   Normalize density:
                $$ \text{density} = \begin{cases} \frac{\text{worleyDensity}}{\text{amplitudeTotal}} & \text{if } \text{amplitudeTotal} > 0 \\ 0 & \text{otherwise} \end{cases} $$
                [S1, unofficial]
            *   Apply density threshold: if `density` < `Density Threshold`, set `density` = 0 [S1, unofficial].
        *   **Lighting Calculation**:
            *   Calculate sun transmittance by marching towards the sun from $\mathbf{p}$ (see `MarchLight` below) [S1, unofficial].
            *   Calculate camera transmittance (current ray transmittance) from `densityTotal` [S1, unofficial].
            *   Calculate scattering factor using Henyey-Greenstein:
                $$ \text{scatter} = \frac{1 - g^2}{4\pi (1 + g^2 - 2g \cos \theta)^{3/2}} $$
                where $g = \text{_ScatterFactor}$ and $\cos \theta = \text{dot}(\mathbf{d}, \text{_SunDirection})$ [S1, unofficial].
            *   Accumulate brightness:
                $$ \text{brightness} = \text{brightness} + \text{density} \times \text{sunTransmittance} \times \text{cameraTransmittance} \times \text{scatter} \times \text{stepSize} \times 0.5 $$
                [S1, unofficial]
        *   Update `densityTotal` += `density` $\times$ `stepSize` [S1, unofficial].
        *   Update `distanceTravelled` += `stepSize` [S1, unofficial].

5.  **Calculate Transmittance and Final Color**:
    *   Compute cloud transmittance using Beer-Lambert law:
        $$ T_{cloud} = e^{-\text{densityTotal} \times \text{_AbsorptionCoefficient}} $$
        [S1, unofficial]
    *   Compute cloud color:
        $$ C_{cloud} = \text{brightness} \times \text{_SunColour.rgb} $$
        [S1, unofficial]
    *   Return final color with alpha:
        $$ \text{return } \text{lerp}(C_{bg}, C_{cloud}, 1 - T_{cloud}) $$
        Note: The source code uses `lerp(originalColour, cloudColour, cloudColour.a)` where `cloudColour.a` is `1 - cloudTransmittance` [S1, unofficial].

### Helper: MarchLight (Sun Transmittance)
To calculate how much sunlight reaches the current sample point $\mathbf{p}$, a secondary ray march is performed towards the sun [S1, unofficial]:
1.  Initialize `lightDensity` = 0, `stepSize` = 20.0, `stepLimit` = 200 [S1, unofficial].
2.  Loop `i` from 0 to `stepLimit`:
    *   Advance position: $\mathbf{p}_{sun} = \mathbf{p} + \text{_SunDirection} \times \text{stepSize}$ [S1, unofficial].
    *   Check if $\mathbf{p}_{sun}$ is inside the cloud volume AABB [S1, unofficial].
    *   If inside, sample noise at scaled coordinates and add to `lightDensity` [S1, unofficial].
3.  Return transmittance:
    $$ T_{sun} = e^{-\text{lightDensity} \times \text{_AbsorptionCoefficient}} $$
    [S1, unofficial]

### Parameters
*   `_AbsorptionCoefficient`: Controls light absorption. Higher values darken clouds [S1, unofficial].
*   `_ScatterFactor` ($g$): Controls forward scattering strength. Higher values create stronger silver linings [S1, unofficial].
*   `Density Threshold`: Minimum noise value to contribute to density. Higher values create gaps [S1, unofficial].
*   `Cloud Scale`: World-space frequency of cloud features. Higher values result in smaller clouds [S1, unofficial].
*   `Octave Count`: Number of FBM layers. Higher values increase edge detail [S1, unofficial].
*   `Wind Speed`: Offsets noise coordinates over time to animate clouds [S1, unofficial].

### Edge Cases and Pitfalls
*   **Unnormalized Ray Directions**: If the ray direction is not normalized, the step size relative to the far clip plane becomes incorrect, causing the ray to step past the entire volume instantly. This prevents density gradients from being rendered. The direction must be normalized before intersection tests [S1, unofficial].
*   **Shader Loop Limits**: While loops with dynamic exit conditions can cause shader compilation errors in some environments (e.g., Unity). The implementation uses a `for` loop with a fixed `stepLimit` and an internal `break` condition to ensure stable compilation [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
STEP_SIZE = 20.0
STEP_LIMIT = 64
LIGHT_STEP_LIMIT = 200
LIGHT_STEP_SIZE = 20.0

def render_clouds(original_colour, uv, ray_direction, camera_world_pos, 
                  cloud_min, cloud_max, sun_direction, sun_colour,
                  absorption_coeff, scatter_factor, density_threshold, 
                  cloud_scale, octave_count, wind_offset):
    
    # 1. Retrieve Background
    bg_color = sample_texture(main_tex, uv)
    
    # 2. Initialize Ray
    origin = camera_world_pos
    direction = normalize(ray_direction)
    
    # 3. Ray-Box Intersection
    entry, exit = ray_box_intersection(origin, direction, cloud_min, cloud_max)
    
    if entry > exit:
        return bg_color
    
    # 4. March Density
    brightness = 0.0
    density_total = 0.0
    distance_travelled = 0.0
    ray_pos = origin + direction * entry
    
    for i in range(STEP_LIMIT):
        if distance_travelled > exit:
            break
            
        ray_pos = ray_pos + direction * STEP_SIZE
        distance_travelled += STEP_SIZE
        
        # Procedural Density
        world_scaled_ray = ray_pos * cloud_scale
        # Apply wind offset to world_scaled_ray here if needed
        
        worley_density = 0.0
        amplitude = 1.0
        frequency = 1.0
        amplitude_total = 0.0
        
        for octave in range(octave_count):
            noise_val = sample_noise(world_scaled_ray * frequency) # e.g., Worley
            worley_density += noise_val * amplitude
            amplitude_total += amplitude
            amplitude *= 0.5
            frequency *= 2.0
            
        if amplitude_total > 0:
            density = worley_density / amplitude_total
        else:
            density = 0.0
            
        if density < density_threshold:
            density = 0.0
            
        # Lighting
        sun_transmittance = march_light(ray_pos, cloud_min, cloud_max, sun_direction, absorption_coeff)
        camera_transmittance = exp(-density_total * absorption_coeff)
        
        scatter = henyey_greenstein(direction, sun_direction, scatter_factor)
        
        brightness += density * sun_transmittance * camera_transmittance * scatter * STEP_SIZE * 0.5
        
        density_total += density * STEP_SIZE
        
    # 5. Final Color
    cloud_transmittance = exp(-density_total * absorption_coeff)
    cloud_colour = brightness * sun_colour.rgb
    
    alpha = 1.0 - cloud_transmittance
    final_colour = lerp(bg_color, cloud_colour, alpha)
    
    return final_colour

def ray_box_intersection(origin, direction, min_bound, max_bound):
    # Slab method
    inv_dir = 1.0 / direction
    t_min = (min_bound - origin) * inv_dir
    t_max = (max_bound - origin) * inv_dir
    
    entry = max(max(t_min.x, t_min.y), t_min.z)
    exit = min(min(t_max.x, t_max.y), t_max.z)
    
    return entry, exit

def henyey_greenstein(ray_dir, sun_dir, scatter_factor):
    cos_angle = dot(normalize(ray_dir), normalize(sun_dir))
    g = scatter_factor
    g_sq = g * g
    numerator = 1.0 - g_sq
    denominator = pow(1.0 + g_sq - 2.0 * g * cos_angle, 1.5)
    return numerator / (4.0 * 3.14159 * denominator)

def march_light(ray_pos, cloud_min, cloud_max, sun_dir, absorption_coeff):
    light_density = 0.0
    current_pos = ray_pos
    
    for i in range(LIGHT_STEP_LIMIT):
        current_pos = current_pos + sun_dir * LIGHT_STEP_SIZE
        
        # Check if inside box
        uvw = (current_pos - cloud_min) / (cloud_max - cloud_min)
        if all(0 <= uvw[i] <= 1 for i in range(3)):
            world_scaled = current_pos * cloud_scale
            light_density += sample_noise(world_scaled) * LIGHT_STEP_SIZE
            
    return exp(-light_density * absorption_coeff)
```

## Subtopics

### CPU-side Ray Marching for Volumetric Clouds
This variant performs the ray marching loop on the CPU rather than the GPU. It renders volumetric clouds by iterating through a defined 3D box, using noise for density and opacity parameters [S2, unofficial]. It is called via `renderer.RenderVolumetricClouds(box_min, box_max, density, noise_scale, octaves, opacity)` each frame [S2, unofficial]. Key parameters include `cloud_box_min` (default -100), `cloud_box_max` (default 100), `cloud_density`, `cloud_noise_scale`, `cloud_octaves`, and `cloud_opacity` [S2, unofficial]. The primary pitfall is a severe performance bottleneck due to CPU execution, which is mitigated by using 4x4 pixel striding, though this significantly reduces rendering quality [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \times M)$, where $N$ is the number of pixels and $M$ is the number of steps per ray. The complexity is dominated by the nested loop for FBM evaluation and the secondary light march.
*   **Memory Complexity**: $O(1)$ per pixel, as no 3D texture storage is required for the cloud volume itself. Noise textures may be used, but they are typically small or procedural [S1, unofficial].
*   **Performance**: The GPU implementation is significantly faster than CPU-side marching [S2, unofficial]. However, the secondary light march (`MarchLight`) adds a constant overhead per sample, which can impact performance at high resolutions or with high `stepLimit` values [S1, unofficial].
*   **Strengths**: Infinite resolution, dynamic animation via wind, no storage requirements for cloud geometry, real-time performance on modern GPUs [S1, unofficial].
*   **Limitations**: Simplified lighting model (single scattering approximation via `MarchLight`), potential for banding if noise sampling is not filtered, shader compilation constraints on some platforms [S1, unofficial].
*   **When to use**: Real-time applications requiring dynamic, large-scale cloud systems where pre-baked textures are impractical [S1, unofficial].
*   **When not to use**: Offline rendering requiring high-fidelity multiple scattering, or hardware with very limited shader compute power [S1, unofficial].

## Background
This method is implemented in Unity as the `CloudRendererProcedural` component, attached to the main camera [S1, unofficial]. The lighting model references "Physically Based Sky, Atmosphere and Cloud Rendering in frostbite" by Sebastien Hillaire (EA Frostbite) for the Henyey-Greenstein scattering function [S1, unofficial]. The FBM implementation is adapted from standard techniques described by Inigo Quilez [S1, unofficial].

## Sources
[S1] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S2] lijiemark/floating-islands-generator, https://github.com/lijiemark/floating-islands-generator, fetched 2026-10-27 (unofficial)
