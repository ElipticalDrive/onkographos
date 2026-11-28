# Long-range Ray Marched Fog

Also called: Long range simplified fog, RDR2 fog approach

## Overview
Long-range Ray Marched Fog is a technique for computing volumetric fog and clouds at distances beyond the range of local volume-based solutions (such as froxel buffers) by ray marching through a simplified density field [S1, unofficial]. It solves the performance bottleneck of evaluating complex local lights and numerous fog volumes at long range by restricting integration to a single directional light (the sun) and assuming constant or globally defined density [S1, unofficial]. This method is distinct from near-range froxel approaches, which are faster for local detail but do not scale to infinite distance, and from full global illumination volumetrics, which are too expensive for real-time long-range rendering [S1, unofficial].

## How it works
The method extends a base volumetric fog system (e.g., froxel-based) by initiating a separate ray marching pass for pixels where the view distance exceeds the local fog buffer's maximum range [S1, unofficial].

**Inputs and Outputs:**
*   **Inputs:** Camera ray origin $\mathbf{o}$ and direction $\mathbf{d}$; maximum froxel distance $t_{froxel}$ (or 0 if disabled); step count $N$; default fog density $\rho_{base}$; default fog color $\mathbf{c}_{fog}$; sun direction $\mathbf{l}_{sun}$; sun intensity $I_{sun}$.
*   **Outputs:** Accumulated fog color $\mathbf{c}_{out}$ and opacity $\alpha_{out}$ for the pixel.

**Data Structures:**
*   **Fog Parameters:** A uniform block containing $t_{froxel}$, $N$, $\rho_{base}$, $\mathbf{c}_{fog}$, $\mathbf{l}_{sun}$, and $I_{sun}$ [S1, unofficial].
*   **Ray State:** Current distance $t$, accumulated transmittance $T$, accumulated color $\mathbf{C}_{acc}$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Determine the start distance $t_{start}$. If froxel fog is enabled, $t_{start} = t_{froxel}$. Otherwise, $t_{start} = 0$ [S1, unofficial].
    *   Initialize ray position $t = t_{start}$.
    *   Initialize accumulated transmittance $T = 1.0$.
    *   Initialize accumulated color $\mathbf{C}_{acc} = \mathbf{0}$ [S1, unofficial].
    *   Calculate step size $\Delta t = (t_{max} - t_{start}) / N$, where $t_{max}$ is the far clip plane or a defined long-range limit [S1, unofficial].

2.  **Ray Marching Loop:**
    *   For $i = 0$ to $N-1$:
        *   Evaluate current position $\mathbf{p} = \mathbf{o} + t \mathbf{d}$ [S1, unofficial].
        *   **Density Evaluation:** In the simplified RDR2 approach, density $\rho(\mathbf{p})$ is often treated as constant $\rho_{base}$ or derived from a global height field if specified. For the basic implementation described in the sources, constant density is used [S1, unofficial].
        *   **Transmittance Update:** Calculate optical depth for this step: $\tau = \rho(\mathbf{p}) \Delta t$. Update transmittance: $T \leftarrow T \cdot e^{-\tau}$ [S1, unofficial].
        *   **Lighting Integration:** Since only the sun is considered, calculate the sun's contribution. The light term $L$ is typically $I_{sun} \cdot \max(0, \mathbf{d} \cdot \mathbf{l}_{sun})$ or similar directional scattering approximation [S1, unofficial].
        *   **Color Accumulation:** Add the scattered light to the accumulator: $\mathbf{C}_{acc} \leftarrow \mathbf{C}_{acc} + \mathbf{c}_{fog} \cdot L \cdot (1 - e^{-\tau}) \cdot T_{prev}$, where $T_{prev}$ is the transmittance before this step [S1, unofficial].
        *   **Advance:** $t \leftarrow t + \Delta t$ [S1, unofficial].
        *   **Termination:** If $T < \epsilon$ (e.g., $0.001$), break loop [S1, unofficial].

3.  **Jittering (Optional):**
    *   To improve quality with Temporal Anti-Aliasing (TAA), jitter the ray direction $\mathbf{d}$ slightly or offset the start position $t_{start}$ per frame [S1, unofficial].

4.  **Blending:**
    *   The result $\mathbf{C}_{acc}$ is blended with the background color $\mathbf{C}_{bg}$ using the final transmittance $T$: $\mathbf{C}_{out} = \mathbf{C}_{bg} \cdot T + \mathbf{C}_{acc}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def long_range_ray_marched_fog(camera_origin, camera_direction, froxel_max_dist, 
                                step_count, base_density, fog_color, sun_direction, 
                                sun_intensity, far_clip_dist):
    # 1. Determine start distance
    t_start = froxel_max_dist if froxel_max_dist > 0 else 0.0
    
    # 2. Initialize ray state
    t = t_start
    transmittance = 1.0
    accumulated_color = vec3(0.0)
    
    # Step size
    step_size = (far_clip_dist - t_start) / step_count
    
    # 3. Marching loop
    for i in range(step_count):
        # Termination check
        if transmittance < 0.001:
            break
            
        # Current position
        pos = camera_origin + camera_direction * t
        
        # Density is constant in the simplified RDR2 approach
        density = base_density
        
        # Optical depth for this step
        optical_depth = density * step_size
        
        # Update transmittance
        prev_transmittance = transmittance
        transmittance *= exp(-optical_depth)
        
        # Sun lighting (simplified directional integration)
        # Assuming single directional light (sun)
        sun_factor = max(0.0, dot(camera_direction, sun_direction))
        light_contribution = sun_intensity * sun_factor
        
        # Accumulate color
        # Scattering term: density * step_size * light * transmittance_in
        scatter = density * step_size * light_contribution * prev_transmittance
        accumulated_color += fog_color * scatter
        
        # Advance ray
        t += step_size
        
    # 4. Blend with background
    # Background color is assumed to be passed or sampled from a skybox
    # For this self-contained function, we return the fog contribution
    # The final pixel color would be: background * transmittance + accumulated_color
    return accumulated_color, transmittance
```

**Parameters:**
*   `step_count`: Number of march steps. Higher values increase quality but cost more performance [S1, unofficial].
*   `base_density`: Constant fog density value [S1, unofficial].
*   `fog_color`: Base color of the fog [S1, unofficial].
*   `froxel_max_dist`: The distance at which the local froxel fog ends. If froxel fog is disabled, this is 0 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Mobile Performance:** Compute shaders are inefficient or unsupported on GLES3/mobile platforms. This method may require disabling froxel fog or using a different post-process approach on these platforms [S1, unofficial].
*   **Lighting Complexity:** Global illumination and local lights must be ignored to maintain performance, especially on mobile. Only the main directional light (sun) is integrated [S1, unofficial].
*   **Fog Volumes:** Local fog volumes should be ignored or kept to a minimum to prevent performance degradation. The method is designed for global, long-range fog [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the step count. Independent of scene geometry complexity, as it does not traverse a BVH or evaluate local volumes [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per pixel, requiring only a few uniform parameters and local accumulators [S1, unofficial].
*   **Performance:** Significantly faster than full volumetric ray marching with local lights and volumes at long range. It is efficient because it avoids complex intersection tests and light sampling [S1, unofficial].
*   **Strengths:** Seamless blending with near-range froxel fog; high performance at long distances; simple implementation [S1, unofficial].
*   **Limitations:** Cannot represent local light scattering or complex fog volume shapes; constant density assumption may look unrealistic in some scenes; quality depends heavily on step count [S1, unofficial].
*   **When to use:** Long-range atmospheric fog and clouds where local detail is not required.
*   **When not to use:** Near-range fog requiring local light interaction, or scenes with many distinct fog volumes [S1, unofficial].

## Background
The method is based on the system used in *Red Dead Redemption 2* (RDR2) for rendering long-range fog and clouds [S1, unofficial]. It was proposed as an extension to the Godot Engine's existing froxel-based volumetric fog system to provide high-quality volumetrics at distances beyond the froxel buffer's range [S1, unofficial]. The approach is particularly noted for its efficiency on mobile platforms when froxel fog is disabled or simplified [S1, unofficial].

## Sources
[S1] Implement ray marched volumetric fog as an extension of the current froxel-based solution, https://github.com/godotengine/godot-proposals/issues/8580, fetched 2026-10-26 (unofficial)
