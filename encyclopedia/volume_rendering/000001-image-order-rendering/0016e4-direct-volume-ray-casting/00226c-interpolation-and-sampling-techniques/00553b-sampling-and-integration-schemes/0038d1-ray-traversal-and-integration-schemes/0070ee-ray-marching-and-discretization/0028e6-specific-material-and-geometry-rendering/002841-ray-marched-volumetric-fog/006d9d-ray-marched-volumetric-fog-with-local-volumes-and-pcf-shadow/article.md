# Ray-marched Volumetric Fog with Local Volumes and PCF Shadows

Also called: HDR height fog, Volumetric Lighting and Height Fog, volumetric direct-light scattering

## Overview
This method renders HDR height fog and volumetric direct-light scattering by integrating exponential height density and up to 16 transformed box volumes along depth-bounded rays [S1, unofficial]. It distinguishes itself from surface-biased approaches by performing volume shadow lookups using hardware PCF at the actual sample point, eliminating light leakage through nearby blockers and removing the cost of repeating raster surface soft-shadow filters at every march step [S1, unofficial]. It operates as a full-resolution, fixed-step, single-scattering pass executed before TAA, bloom, and tone mapping [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Camera parameters (perspective, orthographic, or infinite-far), scene depth buffer, up to 16 local volume definitions (transformed boxes), scene light list (directional, point, spot), and HDR color buffer.
*   **Outputs:** Accumulated fog color and opacity per pixel, integrated into the HDR buffer.

**Data Structures**
*   **Local Volume:** A transformed box defined by a $4 \times 4$ affine transformation matrix and volume properties (extinction coefficient, albedo, scattering phase function parameters). The implementation rejects invalid volume transforms [S1, unofficial].
*   **Light List:** Contains directional, point, and spot lights sharing the scene’s attenuation and shadow maps [S1, unofficial].

**Algorithm Steps**
1.  **Ray Generation:** For each pixel, generate a ray based on the camera type (perspective, orthographic, or infinite-far). Determine the ray's near and far bounds using the scene depth [S1, unofficial].
2.  **Ray Marching:** Discretize the ray into fixed steps. The number of steps is configurable (e.g., 48 steps in reference measurements) [S1, unofficial].
3.  **Sample Point Calculation:** For each step, calculate the sample position at the **segment midpoint** between the current and next step boundary [S1, unofficial].
4.  **Height Density Integration:** Evaluate the exponential height density at the sample point. This contributes to the base extinction along the ray [S1, unofficial].
5.  **Local Volume Intersection:** Check the sample point against all active local volumes (up to 16).
    *   Use ray–box interval tests to determine if the sample lies within a volume.
    *   Preserve thin-volume extinction between march samples by integrating over the ray segment length inside the volume [S1, unofficial].
    *   If the sample is inside a volume, compute the additive extinction and scattering contribution using the volume's properties.
6.  **Lighting Evaluation:** For each light in the scene:
    *   Calculate the light direction and distance to the sample point.
    *   Apply light attenuation.
    *   **Shadow Lookup:** Perform a hardware PCF shadow lookup using the shadow map associated with the light. Crucially, sample the shadow map at the **actual 3D sample point** projected into light space, without applying surface-normal or depth offsets [S1, unofficial]. This prevents light leakage through nearby blockers [S1, unofficial].
    *   Apply the Henyey–Greenstein phase function for scattering towards the camera [S1, unofficial].
7.  **Sanitization:** Ensure all color and albedo values are finite; sanitize non-finite values [S1, unofficial].
8.  **Accumulation:** Integrate the scattered light and accumulated extinction along the ray. The result is added to the HDR buffer [S1, unofficial].

**Formulas**
*   **Exponential Height Density:** $\rho(h) = \rho_0 e^{-h/H}$, where $h$ is height, $\rho_0$ is sea-level density, and $H$ is scale height [S1, unofficial].
*   **Beer-Lambert Attenuation:** $T = e^{-\int \sigma_t(s) ds}$, where $\sigma_t$ is the extinction coefficient [S1, unofficial].
*   **Henyey-Greenstein Phase Function:** $p(\theta) = \frac{1-g^2}{4\pi (1+g^2-2g\cos\theta)^{3/2}}$, where $g$ is the anisotropy factor and $\theta$ is the scattering angle [S1, unofficial].

**Parameters**
*   **Max Local Volumes:** 16 [S1, unofficial].
*   **Steps:** Fixed number (e.g., 48) [S1, unofficial].
*   **Sample Position:** Segment midpoint [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Light Leakage:** Surface-biased shadow lookups cause light leakage through nearby blockers. This is mitigated by using volume shadow lookups at the actual sample point [S1, unofficial].
*   **Invalid Transforms:** Invalid volume transforms must be rejected to prevent artifacts [S1, unofficial].
*   **Non-Finite Values:** Non-finite colors or albedos must be sanitized [S1, unofficial].
*   **TAA History:** Fog has no dedicated history buffer for TAA; it relies on the accumulated scene history, which may cause artifacts if not handled carefully [S1, unofficial].
*   **Blended Surfaces:** Blended surfaces use opaque depth rather than separate per-layer fog distances, which may affect depth-based fog culling [S1, unofficial].
*   **Soft Shadows:** Wide raster soft-shadow filtering does not apply to the fog pass; hardware PCF is used instead [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_fog_pass(camera, depth_buffer, local_volumes, lights, hdr_buffer):
    # Initialize output buffer
    output = zeros_like(hdr_buffer)
    
    # Iterate over all pixels
    for y, x in pixels():
        # Generate ray based on camera type
        ray = camera.generate_ray(x, y)
        
        # Determine depth bounds
        near_z = depth_buffer[x, y]
        far_z = camera.far_plane
        
        # Initialize accumulated color and opacity
        acc_color = vec3(0.0)
        acc_opacity = 0.0
        
        # Ray marching loop
        for step in range(num_steps):
            # Calculate segment start and end in world space
            t_start = ray.origin + ray.direction * (step * step_size)
            t_end = ray.origin + ray.direction * ((step + 1) * step_size)
            
            # Sample at midpoint
            t_mid = (t_start + t_end) / 2.0
            sample_pos = ray.origin + ray.direction * t_mid
            
            # Check if sample is within depth bounds
            if sample_pos.z < near_z or sample_pos.z > far_z:
                break
                
            # Calculate extinction for this segment
            extinction = 0.0
            
            # 1. Height Fog Extinction
            height_density = exponential_height_density(sample_pos.y)
            segment_length = length(t_end - t_start)
            extinction += height_density * segment_length
            
            # 2. Local Volume Extinction
            for volume in local_volumes:
                if ray_box_intersection(ray, t_start, t_end, volume.transform):
                    # Calculate intersection interval length within volume
                    interval_length = get_intersection_length(ray, t_start, t_end, volume.transform)
                    extinction += volume.extinction_coefficient * interval_length
            
            # Calculate transmission for this segment
            transmission = exp(-extinction)
            
            # Calculate scattered light
            scattered_light = vec3(0.0)
            for light in lights:
                # Light direction and distance
                light_dir = normalize(light.position - sample_pos)
                light_dist = length(light.position - sample_pos)
                
                # Attenuation
                attenuation = light.attenuation(light_dist)
                
                # Shadow Lookup (Volume-based)
                # Project sample point to light space
                light_space_pos = light.view_matrix * sample_pos
                shadow = hardware_pcf_shadow_lookup(light.shadow_map, light_space_pos)
                
                # Phase function
                phase = heney_greenstein_phase(ray.direction, light_dir, light.anisotropy)
                
                # Add scattered light
                scattered_light += light.color * attenuation * shadow * phase * volume_albedo
                
            # Accumulate
            # Note: This is a simplified single-scattering accumulation
            # In practice, this might be integrated more carefully with Beer-Lambert
            acc_color += scattered_light * transmission
            acc_opacity += (1.0 - transmission) * (1.0 - acc_opacity)
            
        # Write result
        output[x, y] = acc_color
```

## Subtopics

### Ray-marched Volumetric Fog
The parent technique that traces rays through a participating medium to simulate light absorption and scattering. Choose this when you need general volumetric effects like fog, smoke, or clouds. [Ray-marched Volumetric Fog](4248-ray-marched-volumetric-fog.md)

### Ray-marched Volumetric Fog with Local Volumes and PCF Shadows
This specific implementation adds support for discrete, transformed box volumes and uses hardware PCF for shadows at the sample point. Choose this when you need efficient, artifact-free shadows for local volumetric objects like fog planes or light shafts. [Ray-marched Volumetric Fog with Local Volumes and PCF Shadows](4249-ray-marched-volumetric-fog-with-local-volumes-and-pcf-shadows.md)

## Operating characteristics
*   **Time Complexity:** $O(P \cdot S \cdot (L + V))$, where $P$ is pixels, $S$ is steps, $L$ is lights, and $V$ is local volumes. Direct-light evaluation visits every light at each step without clustered culling [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for storing local volume data, plus standard shadow map memory [S1, unofficial].
*   **Performance:** On Apple M3 Max, 640×480 resolution, 48 steps, 60 timed frames: 4.18 ms/frame with fog, 2.68 ms disabled [S1, unofficial]. The previous surface-shadow lookup measured 8.36 ms with fog, showing a significant performance improvement [S1, unofficial].
*   **Strengths:** Eliminates light leakage from nearby blockers; removes the cost of repeating raster soft-shadow filters; supports various camera types and HDR post-processing [S1, unofficial].
*   **Limitations:** Single scattering only; no dedicated TAA history for fog; scaling with lights and volumes can be expensive without culling; wide raster soft-shadow filtering is not applicable [S1, unofficial].
*   **When to Use:** When high-quality, artifact-free volumetric shadows are needed for local fog volumes and height fog, and performance is acceptable for the target resolution and light count [S1, unofficial].
*   **When Not to Use:** When multiple scattering is required; when the scene has a very large number of lights and volumes without culling; when raster soft-shadow filtering is preferred for performance [S1, unofficial].

## Background
This method was implemented in ParadiseEngine, as documented in `src/Paradise.Rendering.Pbr/FOG.md` [S1, unofficial]. It was introduced in pull request #278, authored by `quabug` and reviewed by `opencara` in September 2026 [S1, unofficial]. The implementation was validated with 110 passed tests in the full PBR suite and 15 focused fog tests, covering Beer-Lambert attenuation, height falloff, depth clipping, and various volume types [S1, unofficial]. It is enabled via scene settings and the `rendering.fog` engine switch [S1, unofficial].

## Sources
[S1] Add volumetric lighting and height fog, https://github.com/ParadiseEngine/ParadiseEngine/pull/278, fetched 2026-10-29 (unofficial)
