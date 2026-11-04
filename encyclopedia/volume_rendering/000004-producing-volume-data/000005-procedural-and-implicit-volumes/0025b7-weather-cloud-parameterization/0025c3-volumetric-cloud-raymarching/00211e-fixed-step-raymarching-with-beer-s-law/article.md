# Fixed-Step Raymarching with Beer's Law

Also called: Fixed-Step Raymarcher, MarchDensity, Volumetric Cloud Density Marching, Volumetric Cloud Renderer

## Overview
Fixed-Step Raymarching with Beer's Law is a real-time volumetric rendering technique that simulates cloud-like structures by tracing rays through a 3D density field and integrating light attenuation using Beer's Law [S1, S2, unofficial]. It differs from its parent [Volumetric Cloud Raymarching](1248-volumetric-cloud-raymarching.md) by explicitly decoupling the density sampling (often via Worley and Perlin noise) from the radiative transfer calculation, applying a fixed step size for deterministic traversal rather than adaptive stepping [S1, S2, unofficial]. It differs from its child [Volumetric Ray Marching with Noise](2562-volumetric-ray-marching-with-noise.md) by relying on scalar density thresholds and Beer's Law for transmittance, whereas the child uses normalized noise vectors to modulate density and light accumulation directly [S1, unofficial].

## How it works
**Inputs:**
- Camera position ($P_{cam}$), ray direction ($\vec{d}$) derived from UV coordinates.
- Cloud volume bounding box (AABB) defined by $\vec{B}_{min}$ and $\vec{B}_{max}$ [S1, S2, unofficial].
- Sun direction ($\vec{S}_{dir}$) and Sun color ($C_{sun}$) [S2, unofficial].
- Noise textures or procedural functions for Worley ($N_W$) and Perlin ($N_P$) noise [S1, S2, unofficial].
- Parameters: $STEP\_SIZE$, $LIGHT\_STEP\_SIZE$, $MAX\_STEPS$, $uUserDensity$, $\vec{uUserScale}$, $\_DensityThreshold$, $\_AbsorptionCoefficient$, $\_ScatterFactor$ [S1, S2, unofficial].

**Outputs:**
- Final pixel color ($C_{final}$) and alpha/transmittance [S1, S2, unofficial].

**Data Structures:**
- `ray`: Struct containing origin ($\vec{o}$) and direction ($\vec{d}$) [S1, unofficial].
- `hitData`: Struct containing entry distance ($t_{entry}$) and exit distance ($t_{exit}$) from AABB intersection [S1, unofficial].

**Steps:**

1.  **Ray Initialization:**
    Calculate the view ray direction $\vec{d}$ from the camera position and screen UV coordinates. Apply any camera rotation matrix if required [S1, unofficial].

2.  **AABB Intersection:**
    Intersect the view ray with the cloud volume AABB.
    - If no intersection, return the sky color [S1, unofficial].
    - Let $t_{entry}$ and $t_{exit}$ be the distances to the entry and exit points along the ray [S1, S2, unofficial].
    - Calculate the total distance to march: $D_{total} = t_{exit} - t_{entry}$ [S1, unofficial].
    - Set initial ray position: $\vec{p}_{start} = \vec{o} + \vec{d} \cdot t_{entry}$ [S2, unofficial].
    - *Edge Case:* If the camera is inside the volume, $t_{entry}$ may be negative or zero; the march starts from the camera position or the calculated entry point [S1, unofficial].

3.  **Initialization:**
    - Initialize `densityTotal` = 0.0 [S2, unofficial].
    - Initialize `brightness` = 0.0 [S2, unofficial].
    - Initialize `globalTransmittance` = 1.0 [S1, unofficial].
    - Initialize `cloudColor` = $(0, 0, 0)$ [S1, unofficial].
    - Initialize `distanceTravelled` = $t_{entry}$ [S2, unofficial].

4.  **Main Raymarch Loop:**
    Loop for $i$ from 0 to `stepLimit` (exclusive) [S2, unofficial]:
    - Calculate current march point: $\vec{p}_{march} = \vec{o} + \vec{d} \cdot \text{distanceTravelled}$ [S2, unofficial].
    - *Check Exit:* If `distanceTravelled` > $t_{exit}$, break loop [S2, unofficial].
    - *Check Max Steps:* If $i \ge MAX\_STEPS$, break loop [S1, unofficial].

    4.1. **Density Sampling:**
    - Calculate world-scaled position: $\vec{p}_{world} = \vec{p}_{march} \cdot \_CloudScale$ [S2, unofficial].
    - Sample Worley noise $N_W$ and Perlin noise $N_P$ at $\vec{p}_{world}$ [S1, S2, unofficial].
      - *Note:* In some implementations, Worley noise is accumulated over octaves with halving amplitude and doubling frequency, then normalized [S2, unofficial].
      - *Note:* Perlin noise may be summed over multiple octaves (e.g., frequencies 1, 2, 4, 8, 16, 32) [S1, unofficial].
    - Calculate raw density: $\rho_{raw} = N_W \cdot N_P$ [S2, unofficial].
    - Normalize density: $\rho = \rho_{raw} / uUserDensity$ [S1, S2, unofficial].

    4.2. **Density Thresholding:**
    - If $\rho < 0.05$, skip to next step (continue) [S1, S2, unofficial].
    - If $\rho < \_DensityThreshold$, skip accumulation [S2, unofficial].

    4.3. **Accumulate Density:**
    - `densityTotal` += $\rho \cdot STEP\_SIZE$ [S2, unofficial].

    4.4. **Light Ray Marching (Secondary Ray):**
    - Define light ray origin: $\vec{o}_{light} = \vec{p}_{march}$ [S1, unofficial].
    - Define light ray direction: $\vec{d}_{light} = \text{normalize}(\vec{S}_{dir})$ [S1, S2, unofficial].
    - Intersect light ray with AABB to find max light distance $t_{light\_max}$ [S1, unofficial].
    - Initialize `currentLightRayT` = 1.0 [S1, unofficial].
    - Loop for light step $lS$ from $STEP\_SIZE$ to $MAX\_STEPS$ with increment $LIGHT\_STEP\_SIZE$ [S1, unofficial]:
      - If $lS \ge t_{light\_max}$, break [S1, unofficial].
      - Calculate light march point: $\vec{p}_{light} = \vec{o}_{light} + \vec{d}_{light} \cdot lS$ [S1, unofficial].
      - Sample density $\rho_{light}$ at $\vec{p}_{light}$ (using same noise/scale logic) [S1, unofficial].
      - If $\rho_{light} < 0.05$, continue [S1, unofficial].
      - Update light transmittance: `currentLightRayT` *= $\exp(-\rho_{light} \cdot LIGHT\_STEP\_SIZE)$ [S1, unofficial].

    4.5. **Scattering and Color Accumulation:**
    - Calculate local transmittance: $T_{local} = \exp(-\rho \cdot STEP\_SIZE)$ [S1, unofficial].
    - Calculate scattering term: $S = (1.0 - T_{local}) \cdot \text{currentLightRayT}$ [S1, unofficial].
      - *Alternative Implementation:* Some sources use Henyey-Greenstein phase function $HG(\vec{d}, \vec{S}_{dir}, \_ScatterFactor)$ and Beer's Law explicitly:
        - $T_{cam} = \exp(-\text{densityTotal} \cdot \_AbsorptionCoefficient)$ [S2, unofficial].
        - $S = HG(\vec{d}, \vec{S}_{dir}, \_ScatterFactor)$ [S2, unofficial].
        - Accumulate brightness: `brightness` += $\rho \cdot \text{currentLightRayT} \cdot T_{cam} \cdot S \cdot STEP\_SIZE \cdot 0.5$ [S2, unofficial].
    - Accumulate cloud color: `cloudColor` += `globalTransmittance` * $S$ * $C_{sun}$ [S1, unofficial].
      - *Note:* `globalTransmittance` may be updated as `globalTransmittance` *= $T_{local}$ [S1, unofficial].

    4.6. **Step Increment:**
    - `distanceTravelled` += $STEP\_SIZE$ [S2, unofficial].

5.  **Final Compositing:**
    - Calculate final cloud color: $C_{cloud} = \text{brightness} \cdot C_{sun}$ [S2, unofficial] or $C_{cloud} = \text{cloudColor}$ [S1, unofficial].
    - Calculate final transmittance: $T_{final} = \exp(-\text{densityTotal} \cdot \_AbsorptionCoefficient)$ [S2, unofficial].
    - *Sun Glare:* If the dot product of the view direction and sun direction is $> 0.9995$, mix in a special sun glare color [S1, unofficial].
    - Return $C_{final} = C_{cloud} + C_{sky} \cdot T_{final}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fixed_step_raymarch(ray_origin, ray_direction, box_min, box_max, sun_dir, sun_color, noise_func, params):
    """
    ray_origin: vec3
    ray_direction: vec3
    box_min: vec3
    box_max: vec3
    sun_dir: vec3
    sun_color: vec3
    noise_func: function(vec3) -> float
    params: dict containing STEP_SIZE, LIGHT_STEP_SIZE, MAX_STEPS, uUserDensity, _DensityThreshold, _AbsorptionCoefficient, _ScatterFactor, _CloudScale
    """
    
    # 1. Intersect ray with AABB
    hit = intersect_aabb(ray_origin, ray_direction, box_min, box_max)
    if not hit.hit:
        return sky_color
    
    t_entry = hit.t_min
    t_exit = hit.t_max
    total_dist = t_exit - t_entry
    
    # 2. Initialize accumulators
    density_total = 0.0
    brightness = 0.0
    global_transmittance = 1.0
    cloud_color = vec3(0.0)
    distance_travelled = t_entry
    
    step_size = params['STEP_SIZE']
    light_step_size = params['LIGHT_STEP_SIZE']
    max_steps = params['MAX_STEPS']
    density_threshold = params['_DensityThreshold']
    absorption_coeff = params['_AbsorptionCoefficient']
    cloud_scale = params['_CloudScale']
    user_density = params['uUserDensity']
    
    # 3. Main Loop
    for i in range(max_steps):
        if distance_travelled > t_exit:
            break
            
        # Current position
        march_pos = ray_origin + ray_direction * distance_travelled
        
        # Density Sampling
        world_pos = march_pos * cloud_scale
        density_raw = noise_func(world_pos)
        density = density_raw / user_density
        
        # Thresholding
        if density < 0.05 or density < density_threshold:
            distance_travelled += step_size
            continue
            
        # Accumulate Density
        density_total += density * step_size
        
        # Secondary Light Ray
        light_origin = march_pos
        light_dir = normalize(sun_dir)
        light_hit = intersect_aabb(light_origin, light_dir, box_min, box_max)
        max_light_dist = light_hit.t_max if light_hit.hit else 0.0
        
        light_transmittance = 1.0
        light_t = step_size
        while light_t <= max_steps and light_t < max_light_dist:
            light_pos = light_origin + light_dir * light_t
            light_density_raw = noise_func(light_pos * cloud_scale)
            light_density = light_density_raw / user_density
            
            if light_density >= 0.05:
                light_transmittance *= exp(-light_density * light_step_size)
            
            light_t += light_step_size
            
        # Scattering Calculation (Method 1: Simplified)
        local_transmittance = exp(-density * step_size)
        scattering = (1.0 - local_transmittance) * light_transmittance
        cloud_color += global_transmittance * scattering * sun_color
        
        # Update Global Transmittance
        global_transmittance *= local_transmittance
        
        # Step Increment
        distance_travelled += step_size
        
    # 4. Final Output
    # Method 2 (Alternative from S2) for final color/transmittance calculation:
    # camera_transmittance = exp(-density_total * absorption_coeff)
    # cloud_colour = brightness * sun_color
    # return float4(cloud_colour, 1.0 - camera_transmittance)
    
    # Using Method 1 accumulation:
    final_color = cloud_color + sky_color * global_transmittance
    return final_color
```

## Subtopics

### [Volumetric Ray Marching with Noise](2562-volumetric-ray-marching-with-noise.md)
Uses normalized noise vectors to modulate density and light, differing from the scalar density/Beer's Law approach of this method.

## Operating characteristics
- **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of steps in the main raymarch loop and $M$ is the number of steps in the secondary light raymarch loop per pixel [S1, S2, unofficial].
- **Memory Complexity:** $O(1)$ per pixel, excluding the storage for noise textures if pre-computed [S1, S2, unofficial].
- **Performance:** Real-time capable on modern GPUs. Performance scales with resolution and the number of steps. Higher quality settings increase `stepLimit` and decrease `stepSize`, significantly increasing cost [S2, unofficial].
- **Strengths:** Deterministic step size simplifies implementation and debugging. Beer's Law provides physically plausible attenuation. Noise-based density allows for detailed, organic cloud structures without large 3D textures [S1, S2, unofficial].
- **Limitations:** Fixed step size may lead to aliasing or missing thin cloud details if steps are too large. Requires careful tuning of `STEP_SIZE` and `LIGHT_STEP_SIZE` to balance quality and performance. The secondary light ray adds significant computational overhead per sample [S1, S2, unofficial].
- **When to use:** Real-time cloud rendering in games or interactive applications where performance is critical but visual quality needs to be higher than simple billboard clouds [S1, S2, unofficial].
- **When not to use:** Offline rendering requiring high-fidelity global illumination or very large-scale atmospheric effects where adaptive stepping or pre-computed lighting might be more efficient [S1, S2, unofficial].

## Background
The technique originates from real-time WebGL volumetric cloud rendering projects, specifically those utilizing fixed-step raymarchers to simulate cloud-like structures [S1, unofficial]. It builds upon the principles of volumetric rendering and Beer's Law, adapting them for real-time execution by using procedural noise (Worley and Perlin) for density definition [S1, S2, unofficial]. The specific implementation details, such as the use of Worley and Perlin noise combination and the fixed-step light marching, are documented in unofficial GitHub repositories [S1, S2, unofficial].

## Sources
[S1] M4trixStrike/WebGL-Volumetric-clouds, https://github.com/M4trixStrike/WebGL-Volumetric-clouds, fetched 2026-10-27 (unofficial)
[S2] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
