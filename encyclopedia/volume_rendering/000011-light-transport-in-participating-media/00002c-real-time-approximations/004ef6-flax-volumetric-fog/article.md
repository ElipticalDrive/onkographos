# Flax Volumetric Fog

Also called: Physically Based and Unified Volumetric Rendering in Frostbite, Volumetric fog: Unified, compute shader based solution

## Overview
Flax Volumetric Fog is a standalone, compute-shader-based method for rendering realistic fog and light shafts in real-time, distinct from post-process image-based tricks [S1, unofficial]. It solves the problem of global volumetric lighting by outputting a single 3D texture containing fog color and density for every mapped world position, which can be consumed by both deferred and forward rendering pipelines [S1, unofficial]. It sets itself apart from siblings like [Froxel-based Volumetric Rendering](1919-froxel-based-volumetric-rendering.md) by using a fixed, low-resolution 3D texture aligned to the viewport rather than hierarchical froxels, and from [Voxel Cone Tracing](1454-voxel-cone-tracing.md) by relying on explicit light injection and temporal filtering rather than hardware ray tracing or cone tracing for scattering [S1, unofficial].

## How it works
The algorithm is divided into four sequential steps: Initialize volume properties, Lights Injection, Light Scattering, and Final Integration [S1, unofficial].

### Inputs and Outputs
**Inputs:**
*   Camera viewport dimensions (XY) and view depth range (Z) [S1, unofficial].
*   Light list (position, color, intensity, type) [S1, unofficial].
*   Fog parameters (exponential density coefficients) [S1, unofficial].
*   Shadow maps (optional, for per-light visibility) [S1, unofficial].
*   Previous frame's history buffer (for temporal blending) [S1, unofficial].

**Outputs:**
*   A single 3D texture (`Result.rgb`) storing the final integrated fog color and density per world space position [S1, unofficial].

### Data Structures
The method uses three 3D textures for intermediate storage, aligned to the viewport's XY device coordinates and covering the view frustum's Z depth [S1, unofficial]. Each voxel stores data per world space position [S1, unofficial].

1.  **Scattering/Absorption Texture:**
    *   Format: `R16G16B16A16` (16-bit float per channel) [S1, unofficial].
    *   Layout: `Scattering.rgb` stores scattering color; `Absorption.a` stores absorption color (using the alpha channel) [S1, unofficial].
2.  **Emissive Texture:**
    *   Format: `R11G11B10` (11-bit float for R/G, 10-bit for B) [S1, unofficial].
    *   Layout: `Emissive.rgb` stores emissive color [S1, unofficial].

### Parameters
*   **Fog Density Model:** Exponential fog [S1, unofficial].
    *   Formula: $\rho(d) = e^{-\beta d}$, where $\beta$ is the density coefficient and $d$ is distance [G].
    *   Typical Values: Not given in the sources. Specific parameter values were referenced in a picture not provided in the text [S1, unofficial].
*   **Texture Resolution:** Low resolution, e.g., $150 \times 80 \times 64$ [S1, unofficial].
*   **Temporal Blend Weight:** 7% (0.07) for blending current frame with the previous frame [S1, unofficial].

### Steps

1.  **Initialize Volume Properties**
    Compute shaders write fog attributes into the 3D textures. For each voxel in the frustum-aligned grid, calculate the exponential fog density and store the scattering and absorption colors in the `Scattering/Absorption` texture and emissive data in the `Emissive` texture [S1, unofficial].

2.  **Lights Injection**
    For each light in the scene:
    *   Render the light's contribution into the 3D texture in additive mode using a geometry shader [S1, unofficial].
    *   For each voxel intersected by the light's volume (e.g., sphere for point lights, cone for spot lights):
        *   Calculate visibility. Optionally, sample the per-light shadow map to improve visual quality [S1, unofficial].
        *   Compute the in-scattering phase function value $P(\theta)$ for the angle $\theta$ between the light direction and the view direction [G].
        *   Store the value: $LightColor \times Intensity \times Visibility \times P(\theta)$ into the `Scattering.rgb` texture [S1, unofficial].

3.  **Light Scattering**
    Evaluate light scattering and extinction for each volume cell using the accumulated light data from Step 2 and the fog properties from Step 1 [S1, unofficial]. This step prepares the volumetric coefficients for integration.

4.  **Final Integration**
    A compute shader iterates through the 3D texture and performs final integration along the view ray [S1, unofficial].
    *   **Temporal Filtering:** To reduce aliasing artifacts caused by low resolution and under-sampling (especially with shadow maps), use a temporal Halton sequence to offset samples inside the voxels [S1, unofficial].
    *   **Temporal Blending:** Blend the current frame's result with the previous frame's history buffer using a 7% blend weight [S1, unofficial].
    *   **Fast Camera Handling:** If the camera moves fast and many samples do not hit the history buffer (invalid history), implement super-sampling to smooth the results [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: Initialize Volume Properties
def InitializeVolumeProperties(voxel_grid, fog_params):
    for voxel in voxel_grid:
        # Calculate exponential density based on world position and camera
        distance = length(voxel.world_pos - camera.position)
        density = exp(-fog_params.density_coefficient * distance)
        
        # Store scattering and absorption
        voxel_grid[voxel].scattering = fog_params.scattering_color
        voxel_grid[voxel].absorption = fog_params.absorption_color
        voxel_grid[voxel].emissive = fog_params.emissive_color

# Step 2: Lights Injection
def LightsInjection(voxel_grid, lights, shadow_maps):
    for light in lights:
        # Render light into 3D texture using geometry shader in additive mode
        # Iterate over voxels intersected by light volume
        for voxel in voxels_intersected_by(light.volume):
            # Calculate visibility
            visibility = 1.0
            if light.uses_shadow_maps:
                visibility = sample_shadow_map(light.shadow_map, light.position, voxel.world_pos)
            
            # Calculate phase function (e.g., Henyey-Greenstein or simple cosine)
            # theta is angle between light_dir and view_dir
            phase = calculate_phase_function(light.direction, voxel.view_direction)
            
            # Additive accumulation
            contribution = light.color * light.intensity * visibility * phase
            voxel_grid[voxel].scattering += contribution

# Step 3: Light Scattering (Preparation)
def LightScattering(voxel_grid):
    # Evaluate scattering and extinction per cell
    # This step may combine light injection data with fog properties
    # to prepare coefficients for integration.
    # Specific formula details are not provided in the source.
    pass

# Step 4: Final Integration
def FinalIntegration(voxel_grid, history_buffer, camera, frame_index):
    result_texture = create_3d_texture(voxel_grid.dimensions)
    
    for voxel in voxel_grid:
        # Temporal Jittering using Halton sequence
        jitter = halton_sequence(frame_index, 2) # 2D jitter for XY
        sample_pos = voxel.world_pos + jitter * voxel.size
        
        # Integrate along view ray
        # Note: The source describes running through the 3D texture and integrating
        # along the view ray. This implies a ray-marching step per pixel or voxel.
        # Given the output is a 3D texture, this likely means computing the 
        # transmittance and color for the voxel's contribution to the view ray 
        # passing through it.
        
        transmittance = calculate_transmittance(voxel)
        color = calculate_inscattering(voxel)
        
        current_frame_value = color * transmittance
        
        # Temporal Blending
        if history_buffer.has_valid_data(voxel, camera):
            previous_value = history_buffer.get(voxel)
            blend_weight = 0.07
            final_value = lerp(previous_value, current_frame_value, blend_weight)
        else:
            # Fast Camera Handling: Super-sampling if history is invalid
            # Implementation detail: The source mentions super-sampling to smooth results.
            # We assume a fallback to the current frame or a higher-quality sample.
            final_value = current_frame_value
            
        result_texture[voxel] = final_value
        
    # Store result in history buffer for next frame
    history_buffer.update(result_texture)
    
    return result_texture
```

### Edge Cases and Pitfalls
*   **Aliasing:** Low resolution ($150 \times 80 \times 64$) causes aliasing artifacts, particularly when sampling cascaded shadow maps for the sun with only one sample per cell (under-sampling) [S1, unofficial].
*   **Fast Camera Movement:** When the camera moves fast, many samples may not hit the history buffer, requiring super-sampling to smooth results [S1, unofficial].
*   **Object Size:** The method is better suited for objects with significant size and solid shape due to the low resolution of the 3D texture [S1, unofficial].

## Operating characteristics
*   **Performance:** ~1.1ms on a GeForce 840M laptop, depending on the number of lights and quality settings [S1, unofficial].
*   **Memory:** Uses three 3D textures. Resolution is typically low (e.g., $150 \times 80 \times 64$) to maintain performance [S1, unofficial].
*   **Strengths:** Fully standalone visual feature; async execution during lighting pass; usable by both deferred and forward rendering; reduced edge artifacts and aliasing flickering due to temporal filtering [S1, unofficial].
*   **Limitations:** Low resolution limits detail; requires significant object size for best results; excludes post-fx light shafts (though the effect itself creates light shafts) [S1, unofficial].
*   **When to use:** Games that use fog or light shafts and can afford a global effect with low-resolution 3D textures [S1, unofficial].
*   **When not to use:** Scenes requiring high-frequency volumetric detail or very small-scale fog effects [S1, unofficial].

## Background
The method was presented by Flax Engine in a blog post titled "Flax Facts #14 - Volumetric Fog" [S1, unofficial]. It is described as a standalone visual feature that pushes fog rendering limits using compute shaders and UAVs on GPU [S1, unofficial]. The author notes that while the algorithm is inspired by concepts like those in Frostbite (referenced in aliases), the implementation details provided are specific to Flax Engine [S1, unofficial].

## Sources
[S1] Flax Facts #14 - Volumetric Fog, https://flaxengine.com/blog/flax-facts-14-volumetric-fog/, fetched 2026-10-29 (unofficial)
