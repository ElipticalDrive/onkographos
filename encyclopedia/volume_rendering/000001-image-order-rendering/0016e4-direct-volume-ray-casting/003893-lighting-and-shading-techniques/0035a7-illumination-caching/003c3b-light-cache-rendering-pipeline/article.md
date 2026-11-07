# Light Cache Rendering Pipeline

Also called: GPU Volume Ray-Caster with Light Cache

## Overview
The Light Cache Rendering Pipeline is a direct volume ray-casting technique that pre-computes illumination attenuation factors (ambient occlusion and directional shadows) into a 3D texture, allowing for real-time rendering of complex lighting in volumetric data [S1, unofficial]. It distinguishes itself from standard ray-casting by replacing per-sample shadow ray traversal with single-texture lookups of pre-aggregated extinction coefficients, achieving speedups of over 45 times while maintaining visual quality comparable to dense sampling [S1, unofficial].

## How it works
The method relies on the assumption that volume material scatters light isotropically, allowing the lighting distribution inside the volume to be pre-computed and stored in an illumination cache [Parent]. The pipeline consists of three phases: SAT construction, Light Cache computation, and Ray-casting.

### Inputs and Outputs
*   **Input Volume ($V$):** 3D texture containing extinction coefficients or density values.
*   **Transfer Function ($TF$):** Maps scalar values in $V$ to color and opacity.
*   **Light Sources ($L$):** Positions and intensities of light sources.
*   **Output:** Rendered image.
*   **Intermediate Data:**
    *   **Light Cache Texture ($T$):** 3D texture storing summation terms for ambient occlusion (AO) and directional shadows in different channels.
    *   **SAT Texture ($S$):** 3D Summed Area Table texture constructed from the transfer-function-applied volume.

### Data Structures
*   **Light Cache Texture ($T$):** A 3D texture where each voxel stores pre-computed attenuation factors. The specific channel layout is not detailed in the sources, but it contains separate terms for AO/color bleeding and directional shadows [S1, unofficial].
*   **SAT Texture ($S$):** A 3D texture representing the Summed Area Table of the volume data after applying the transfer function. It supports $O(1)$ aggregate queries over rectangular cuboids [S1, unofficial].

### Algorithm Steps

1.  **Initialization and Change Detection:**
    *   Determine if the Transfer Function ($TF$) or light positions have changed since the last frame [S1, unofficial].
    *   If $TF$ has changed, recompute the entire light cache. If only light positions changed, recompute only directional shadows [S1, unofficial].

2.  **Transfer Function Application:**
    *   Apply $TF$ to volume $V$ to produce a texture $T_{temp}$ containing extinction coefficients and color information [S1, unofficial].

3.  **SAT Construction:**
    *   Compute the 3D Summed Area Table ($S$) from $T_{temp}$ using recursive doubling with ping-pong textures [S1, unofficial].
    *   The SAT allows efficient computation of the sum of extinction coefficients over any voxel neighborhood $\Omega$:
        $$ \sum_{\Omega} \Delta t \cdot \tau_j $$
        where $\tau_j$ is the extinction coefficient and $\Delta t$ is the step size [S1, unofficial].

4.  **Ambient Occlusion / Color Bleeding Computation:**
    *   For each voxel in $S$:
        *   Sample shells (spherical neighborhoods) from $S$ using aggregate queries [S1, unofficial].
        *   Sum the weighted extinction coefficients from these shells [S1, unofficial].
        *   Store the result in the AO channel of $T$ [S1, unofficial].

5.  **Directional Shadow Computation (if lights changed):**
    *   For each light source $L$:
        *   For each voxel in $S$:
            *   Query cuboids towards light source $L$ using the SAT [S1, unofficial].
            *   Sum the weighted extinction coefficients along the path [S1, unofficial].
            *   Store the result in the shadow channel of $T$ corresponding to $L$ [S1, unofficial].

6.  **Ray-Casting Pass:**
    *   For each pixel on the screen:
        *   Compute entry and exit points for volume $V$ [S1, unofficial].
        *   Compute ray $R$ from entry and exit points [S1, unofficial].
        *   For each sample position $P$ along $R$:
            *   Lookup volume $V$ and apply $TF$ to get color and opacity [S1, unofficial].
            *   Lookup texture $T$ at $P$ to retrieve pre-computed AO and shadow factors [S1, unofficial].
            *   Evaluate illumination model using the retrieved factors [S1, unofficial].
            *   Add contribution to the pixel [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def light_cache_rendering_pipeline(volume_V, transfer_function, light_sources, screen_resolution):
    """
    Main rendering pipeline for Light Cache method.
    """
    # State tracking
    tf_changed = True  # Assume true for first frame
    lights_changed = True  # Assume true for first frame
    
    # Textures
    # T stores light cache (AO and shadows)
    # S stores SAT
    # T_temp stores TF-applied volume
    
    for frame in range(num_frames):
        # 1. Check for changes
        if transfer_function_changed():
            tf_changed = True
            lights_changed = True # TF change invalidates all cache
        if light_positions_changed():
            lights_changed = True
            
        # 2. Apply Transfer Function if TF changed
        if tf_changed:
            T_temp = apply_transfer_function(volume_V, transfer_function)
            
            # 3. Compute SAT from T_temp
            # Using recursive doubling with ping-pong textures
            S = compute_sat_recursive_doubling(T_temp)
            
            # 4. Compute Ambient Occlusion / Color Bleeding
            # Initialize T with zeros
            T = initialize_texture_zeros()
            
            for voxel in S:
                # Sample shells from S
                shells = sample_spherical_shells(S, voxel.position)
                # Sum weighted shells
                ao_factor = sum_weighted_extinction(shells)
                # Store to T (AO channel)
                T[voxel.position].ao = ao_factor
            
            tf_changed = False
            
        # 5. Compute Directional Shadows if lights changed
        if lights_changed:
            # Ensure T exists and has space for shadow channels
            # Re-initialize shadow channels only if needed, or overwrite
            for light in light_sources:
                for voxel in S:
                    # Query cuboids towards light source L
                    # This aggregates extinction along the path from voxel to light
                    shadow_factor = query_cuboid_shadow(S, voxel.position, light.position)
                    # Store sum to T (shadow channel for light)
                    T[voxel.position].shadow[light.id] = shadow_factor
            lights_changed = False
            
        # 6. Ray-Casting Pass
        image = create_image(screen_resolution)
        
        for pixel in screen_resolution:
            # Compute entry and exit point for volume V
            entry, exit = compute_ray_bounds(volume_V, pixel)
            
            # Compute ray R
            ray_direction = normalize(exit - entry)
            ray_origin = entry
            
            accumulated_color = vec3(0.0)
            accumulated_opacity = 0.0
            
            # Sample along ray
            for sample_pos in sample_ray(ray_origin, ray_direction, entry, exit):
                # Lookup volume V and apply transfer function
                vol_value = lookup_volume(volume_V, sample_pos)
                color, opacity = apply_transfer_function_sample(vol_value, transfer_function)
                
                # Lookup texture T
                ao_factor = lookup_light_cache(T, sample_pos, "ao")
                shadow_factors = lookup_light_cache(T, sample_pos, "shadow")
                
                # Evaluate illumination model
                # Note: Exact illumination formula not provided in sources,
                # but uses pre-computed factors [S1, unofficial]
                light_intensity = compute_light_intensity(light_sources, sample_pos, shadow_factors)
                final_color = color * light_intensity * ao_factor
                
                # Volume compositing (front-to-back)
                # Standard DVE formula: C_out = C_in + C_sample * alpha_sample * exp(-sum_extinction)
                # The light cache factors modify the light contribution term
                accumulated_color += final_color * opacity * (1.0 - accumulated_opacity)
                accumulated_opacity += opacity * (1.0 - accumulated_opacity)
                
                if accumulated_opacity > 0.99:
                    break
            
            image[pixel] = accumulated_color
            
    return image

def compute_sat_recursive_doubling(texture):
    """
    Computes 3D SAT using recursive doubling with ping-pong textures.
    """
    # Implementation detail: Standard 3D SAT algorithm
    # 1. Sum along X axis
    # 2. Sum along Y axis
    # 3. Sum along Z axis
    # Using ping-pong buffers to minimize memory bandwidth
    pass

def sample_spherical_shells(sat_texture, position):
    """
    Samples shells (spherical neighborhoods) from SAT.
    Uses aggregate queries to sum extinction coefficients.
    """
    # Aggregate query q over spherical region
    pass

def query_cuboid_shadow(sat_texture, voxel_pos, light_pos):
    """
    Queries cuboids towards light source L.
    Computes sum of extinction coefficients along the path.
    """
    # Aggregate query over the cuboid defined by voxel and light
    pass
```

### Parameters and Edge Cases
*   **Transfer Function Change:** If the TF changes, the entire light cache must be recomputed, including SAT and AO [S1, unofficial].
*   **Light Position Change:** If only light positions change, only the directional shadow factors need recomputation [S1, unofficial].
*   **Precision:** The method relies on the order independence of the summation of exponential extinction coefficients [S1, unofficial].
*   **Homogeneous Media:** The method is particularly effective for highly homogeneous media such as smoke or translucent materials [S1, unofficial].

## Operating characteristics
*   **Performance:** The method yields results similar in quality to dense sampling but is much faster. Specifically, aggregate sampling is over 45 times faster than individually sampled neighborhoods (0.22s vs 10s for ambient occlusion) [S1, unofficial].
*   **Memory:** Requires storage for the Light Cache Texture ($T$) and SAT Texture ($S$), in addition to the volume data. The SAT construction uses ping-pong textures, requiring temporary memory [S1, unofficial].
*   **Strengths:** Real-time rendering of complex lighting effects (AO, soft shadows) in volumes. High speed due to aggregate queries.
*   **Limitations:** Requires isotropic scattering assumption. Recomputation cost if TF changes. Quality depends on the accuracy of the SAT and shell/cuboid approximations.
*   **When to use:** Real-time volume rendering where lighting effects are important and the volume data is static or changes infrequently. Suitable for homogeneous media.
*   **When not to use:** Highly anisotropic scattering. Frequent TF changes.

## Background
The method is based on the work of [S1, unofficial], which presents an extinction-based shading and illumination technique for GPU volume ray-casting. It builds on the concept of illumination caching [Parent] and uses 3D Summed Area Tables for efficient aggregate queries. The implementation is built using OpenGL and GLSL shaders [S1, unofficial].

## Sources
[S1] Extinction-based Shading and Illumination in GPU Volume Ray-Casting, https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf, fetched 2026-10-30 (unofficial)
