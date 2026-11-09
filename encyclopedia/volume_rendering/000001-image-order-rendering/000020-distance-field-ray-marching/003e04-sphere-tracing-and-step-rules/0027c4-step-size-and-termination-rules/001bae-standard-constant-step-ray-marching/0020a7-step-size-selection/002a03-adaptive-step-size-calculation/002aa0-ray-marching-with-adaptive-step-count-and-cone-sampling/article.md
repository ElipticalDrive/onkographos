# Ray Marching with Adaptive Step Count and Cone Sampling

Also called: Adaptive Step Count with Cone Sampling

## Overview
This method renders volumetric clouds by adaptively selecting the number of ray march steps based on the viewing angle and using cone sampling to approximate light transport within the volume [S1, unofficial]. It balances rendering performance and visual quality by using fewer steps for top-down views and more steps for horizon views, while cone sampling mitigates artifacts from single-sample lighting [S1, unofficial]. It differs from standard constant step ray marching by varying the step count dynamically and from sibling methods by using a fixed kernel of 6 offset vectors rotated towards the light source for density estimation along the light path [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- Ray origin $O$ and direction $D$ (normalized) [S1, unofficial].
- Ray segment bounds $t_{start}$ and $t_{end}$ in world space [S1, unofficial].
- Pixel identifier (integer) for jittering [S1, unofficial].
- Time variable $T$ for wind animation [S1, unofficial].
- Light direction $L$ (normalized) [S1, unofficial].
- Earth center position $C_{earth}$ [S1, unofficial].
- Parameters: `baseDensityFactor`, `jitterfactor`, `eccentricity`, `silver_intensity`, `silver_spread`, `brightness` [S1, unofficial].

**Outputs:**
- Accumulated color $C_{out}$ (RGB) [S1, unofficial].
- Accumulated density $\alpha_{accum}$ (scalar, clamped to 1.0) [S1, unofficial].

### Data Structures
The method uses a fixed kernel of 6 offset vectors defined in local space and rotated to align with the light direction [S1, unofficial].
- **Kernel Offsets (Local):**
  1. $(0.1, 0.25, -0.15)$
  2. $(0.2, 0.5, 0.2)$
  3. $(-0.2, 0.1, -0.1)$
  4. $(-0.05, 0.75, 0.05)$
  5. $(-0.1, 1.0, 0.0)$
  6. $(0.0, 3.0, 0.0)$ [S1, unofficial]

### Algorithm Steps
1. **Calculate Adaptive Step Count:** Compute the dot product of the ray direction $D$ and the world up vector $U = (0, 1, 0)$ [S1, unofficial]. Calculate the number of steps $N_{steps}$ using linear interpolation between 35 and 60 steps based on the viewing angle [S1, unofficial]. Top-down views ($d \approx 1$) use 35 steps; horizon views ($d \approx 0$) use 60 steps [S1, unofficial].
   $$ d = \text{dot}(D, U) $$
   $$ N_{steps} = \lfloor \text{mix}(35.0, 60.0, 1.0 - d) \rfloor $$ [S1, unofficial]

2. **Initialize Variables:** Compute the step size $\Delta t = (t_{end} - t_{start}) / N_{steps}$ [S1, unofficial]. Initialize transmittance $T_{trans} = 1.0$ and accumulated density $\alpha_{accum} = 0.0$ [S1, unofficial]. Initialize output color $C_{out} = (0, 0, 0)$ [S1, unofficial].

3. **Construct Light Rotation Matrix (TBN):** Find the axis with the largest absolute component in $L$, denoted $V_{max}$ [S1, unofficial]. Compute $Z_{comp} = \text{cross}(L, V_{max})$ and $X_{comp} = \text{cross}(Z_{comp}, L)$ [S1, unofficial]. Form the rotation matrix $M_{sun}$ with columns $X_{comp}$, $L$, and $Z_{comp}$ [S1, unofficial].
   $$ M_{sun} = \begin{bmatrix} X_{comp} & L & Z_{comp} \end{bmatrix} $$ [S1, unofficial]

4. **Precompute Cone Sampling Offsets:** Transform the 6 local kernel offsets $K_{local}[i]$ into world space aligned with the light: $K_{world}[i] = M_{sun} \times K_{local}[i]$ for $i \in [0, 5]$ [S1, unofficial].

5. **Ray Marching Loop:** Iterate $j$ from $0$ to $N_{steps} - 1$ [S1, unofficial].
   a. **Calculate Sample Position:** Compute current $t = t_{start} + j \cdot \Delta t$ [S1, unofficial]. Generate a jitter offset using `pixelID` and $t$ to select an index modulo 16, then retrieve `jitterLocation` [S1, unofficial]. Update position $P$:
      $$ P = O + t \cdot (D + (\text{jitterLocation}.x, (\text{jitterLocation}.x + \text{jitterLocation}.y) \cdot \text{jitterfactor}, \text{jitterLocation}.y)) $$ [S1, unofficial]
   b. **Sample Density:** Transform $P$ to relative atmospheric coordinates and divide by 8.0 to control noise frequency [S1, unofficial]. Skew the sample point using wind direction and time $T$ [S1, unofficial]. Calculate relative height $h$ from $P$ and $C_{earth}$ [S1, unofficial]. Sample low-frequency density: $\rho_{base} = \text{SampleLowFreq}(P_{skewed}, P, h, C_{earth}) \cdot \text{baseDensityFactor}$ [S1, unofficial]. If $\rho_{base} > 0.0$, erode with high-frequency noise: $\rho_{high} = \text{ErodeCloud}(\rho_{base} \cdot 1.4, D, P_{skewed}, h)$ [S1, unofficial]. Update accumulated density: $\alpha_{accum} \leftarrow \alpha_{accum} + \rho_{high} \cdot 0.5$ [S1, unofficial]. Check for early termination: if $\alpha_{accum} \ge 1.0$, set $\alpha_{accum} = 1.0$ and break the loop [S1, unofficial].
   c. **Cone Sampling for Lighting:** Initialize `densityAlongLight` = 0.0 [S1, unofficial]. Loop $i$ from $0$ to $5$ [S1, unofficial]. Calculate light sample position: $P_{light} = P + \Delta t \cdot K_{world}[i] \cdot i$ [S1, unofficial]. Sample density at $P_{light}$ [S1, unofficial]. If density $> 0.0$, erode with high-frequency noise and add to `densityAlongLight` [S1, unofficial].
   d. **Compute Light Energy and Update Color:** Calculate total light energy using `GetLightEnergy` with parameters: relative height $h$, `densityAlongLight`, $\rho_{base}$, Henyey-Greenstein phase function value $p(\theta)$ with eccentricity `eccentricity`, cosine of the angle between light and view direction, step size $\Delta t$, and brightness multiplier `brightness` [S1, unofficial]. Update transmittance and color:
      $$ T_{trans} = \text{mix}(T_{trans}, E_{light}, 1.0 - \alpha_{accum}) $$ [S1, unofficial]
      $$ C_{out} \leftarrow C_{out} + T_{trans} $$ [S1, unofficial]

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_adaptive_cone(ray_origin, ray_dir, t_start, t_end, pixel_id, time, light_dir, earth_center, params):
    # 1. Adaptive step count
    dot_val = dot(ray_dir, vec3(0.0, 1.0, 0.0))
    max_steps = floor(mix(35.0, 60.0, 1.0 - dot_val))
    step_size = (t_end - t_start) / max_steps
    
    # 2. Initialize
    transmittance = 1.0
    accum_density = 0.0
    return_color = vec3(0.0, 0.0, 0.0)
    
    # 3. TBN matrix for light
    max_comp_idx = argmax(abs(light_dir))
    max_comp_unit = vec3([1.0 if i == max_comp_idx else 0.0 for i in range(3)])
    z_comp = cross(light_dir, max_comp_unit)
    x_comp = cross(z_comp, light_dir)
    sun_rot_matrix = mat3(x_comp, light_dir, z_comp)
    
    # 4. Precompute cone offsets
    local_offsets = [
        vec3(0.1, 0.25, -0.15), vec3(0.2, 0.5, 0.2), vec3(-0.2, 0.1, -0.1),
        vec3(-0.05, 0.75, 0.05), vec3(-0.1, 1.0, 0.0), vec3(0.0, 3.0, 0.0)
    ]
    noise_kernel = [sun_rot_matrix @ k for k in local_offsets]
    
    # 5. Marching loop
    for step in range(max_steps):
        t = t_start + step * step_size
        
        # Jitter
        jitter_idx = hash(pixel_id, t) % 16
        jitter_loc = get_jitter(jitter_idx) # [G] Standard lookup/pseudo-random retrieval
        jitter_offset = vec3(jitter_loc.x, (jitter_loc.x + jitter_loc.y) * params.jitterfactor, jitter_loc.y)
        pos = ray_origin + t * (ray_dir + jitter_offset)
        
        # Density sampling
        rel_coords = transform_to_atmosphere(pos) / 8.0
        skewed_point = skew_with_wind(rel_coords, time)
        rel_height = compute_relative_height(pos, earth_center)
        
        base_density = sample_low_frequency(skewed_point, pos, rel_height, earth_center) * params.baseDensityFactor
        
        if base_density > 0.0:
            high_freq_density = erode_cloud_with_high_frequency(base_density * 1.4, ray_dir, skewed_point, rel_height)
            accum_density += high_freq_density * 0.5
            
        if accum_density >= 1.0:
            accum_density = 1.0
            break
            
        # Cone sampling
        density_along_light = 0.0
        for i in range(6):
            light_pos = pos + step_size * noise_kernel[i] * float(i)
            light_base = sample_low_frequency(light_pos, light_pos, rel_height, earth_center)
            if light_base > 0.0:
                light_high = erode_cloud_with_high_frequency(light_base * 1.4, ray_dir, light_pos, rel_height)
                density_along_light += light_high
                
        # Light energy & update
        cos_angle = dot(ray_dir, light_dir)
        hg_value = heney_greenstein(cos_angle, params.eccentricity, params.silver_intensity, params.silver_spread)
        total_light_energy = get_light_energy(rel_height, density_along_light, base_density, hg_value, cos_angle, step_size, params.brightness)
        
        transmittance = mix(transmittance, total_light_energy, 1.0 - accum_density)
        return_color += transmittance
        
    return return_color
```

### Parameters
- `baseDensityFactor`: Controls cloud density center intensity. Default: 0.380 [S1, unofficial].
- `jitterfactor`: Multiplier for jitter offset in Y component. Default: 1.180 [S1, unofficial].
- `eccentricity`: Henyey-Greenstein scattering parameter. Default: 0.6 [S1, unofficial].
- `silver_intensity`: Parameter for modified Henyey-Greenstein function. Default: 0.7 [S1, unofficial].
- `silver_spread`: Parameter for modified Henyey-Greenstein function. Default: 0.1 [S1, unofficial].
- `brightness`: Lighting brightness multiplier. Default: 5.0 [S1, unofficial].

### Edge Cases and Pitfalls
- **Precision:** The mixing of transmittance and light energy requires careful handling of floating-point precision to avoid banding, especially at low densities [S1, unofficial].
- **Boundaries:** The ray segment bounds $t_{start}$ and $t_{end}$ must correctly clip against the volume boundaries (e.g., sphere intersection with atmosphere) [S1, unofficial].
- **Degenerate Input:** If the ray direction is parallel to the world up vector, the dot product is 1 or -1 [S1, unofficial]. The formula handles $d \approx 1$ (top-down) well [S1, unofficial]. If $d$ is negative (viewing from below), the mix parameter $1.0-d$ exceeds 1, potentially exceeding 60 steps [S1, unofficial]. The source specifies "horizon views" for $d$ near 0, implying the method is optimized for viewing from above or the horizon [S1, unofficial].
- **Kernel Alignment:** The TBN matrix construction assumes $L$ is not parallel to the chosen `maxComp` axis [S1, unofficial]. If $L$ is parallel, the cross product becomes zero [S1, unofficial]. A fallback axis selection is not explicitly detailed in the source but is a standard implementation requirement [G].

## Operating characteristics

### Time and Memory Complexity
- **Time Complexity:** $O(N_{steps} \cdot K)$, where $N_{steps}$ is the adaptive step count (35-60) and $K=6$ is the number of cone samples [S1, unofficial]. The complexity is linear with respect to the number of steps and constant with respect to the scene size (assuming texture lookups are $O(1)$) [G].
- **Memory Complexity:** $O(1)$ auxiliary memory per pixel, excluding the noise texture/sampler state [G].

### Measured Performance
- Specific performance metrics (FPS, resolution) are not provided in the sources [S1, unofficial].

### Scaling
- The method scales well with resolution as it is a per-pixel operation [G]. The adaptive step count reduces computational load for top-down views, providing a performance benefit in scenes with significant vertical viewing angles [S1, unofficial].

### Strengths and Limitations
- **Strengths:** Reduces artifacts from single-sample lighting via cone sampling [S1, unofficial]. Adaptive steps optimize performance based on viewing angle [S1, unofficial].
- **Limitations:** Fixed kernel size (6 samples) may not capture complex light transport in very dense volumes [S1, unofficial]. The method relies on procedural noise, which may lack the detail of texture-based approaches [S1, unofficial].

### When to Use
- Use for real-time volumetric cloud rendering where performance is critical and visual quality needs to be maintained across varying camera angles [S1, unofficial].

### When Not to Use
- Not suitable for offline rendering where higher fidelity light transport (e.g., multiple scattering) is required [G]. Not suitable for scenes with dynamic light sources that change rapidly, as the TBN matrix must be recomputed per ray [G].

## Background

### Origin
The method is derived from the implementation in the open-source project "realtime-volumetric-cloudscapes" by erickTornero [S1, unofficial].

### History
The technique combines adaptive step sizing, which is common in ray marching to optimize performance, with cone sampling, a technique used to approximate the integration of light through a volume by sampling along a cone aligned with the light direction [S1, unofficial].

### Usage
The method is used in real-time graphics applications, specifically for rendering atmospheric clouds in games or simulations [S1, unofficial].

## Sources

[S1] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
