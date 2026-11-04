# Froxel-based Volumetric Rendering

Also called: Froxel Rendering, Froxel-based volumetric fog and light shafts, Godot Volumetric Rendering Pipeline, Wronski/Frostbite-style frustum-aligned volumetric fog, froxels

## Overview
Froxel-based Volumetric Rendering is a real-time approximation for light transport in participating media that amortizes ray-queried occlusion lighting work by dividing the camera view frustum into a 3D grid of voxels ("froxels") [S2, unofficial]. It solves the high computational cost of per-pixel ray marching by collapsing per-pixel fog cost to a fixed march in an integrate pass, delivering spatial-accumulation savings [S2, unofficial]. Unlike [Voxel Cone Tracing](1454-voxel-cone-tracing.md) which relies on cone tracing for indirect lighting, or [Precomputed Lighting (Baking)](2169-precomputed-lighting-baking.md) which uses static data, this method computes dynamic in-scattered radiance and shadows per froxel, often combined with [Froxel TAA Reprojection](1920-froxel-taa-reprojection.md) to stabilize temporal noise [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera frustum parameters (near/far planes), tile size, depth slice count, fog density parameters, phase function parameters, albedo, noise parameters, and scene geometry (via TLAS for ray queries) [S2, unofficial].
*   **Outputs:** A 3D texture containing integrated volumetric radiance and density, which is sampled during the final composite pass to blend fog with the scene [S1, unofficial].

**Data Structures:**
*   **Froxel Grid:** A frustum-aligned 3D grid. The XY dimensions correspond to a screen tile footprint, and the Z dimension uses an exponential depth distribution [S2, unofficial].
*   **Volume Buffer:** A 16-bit RGBA 3D texture. RGB channels store wavelength-dependent in-scattered radiance; the Alpha/W channel stores participating media density or extinction [S1, unofficial][S2, unofficial].
*   **History Buffer:** A 3D texture storing the TAA-resolved radiance and density from the previous frame [S1, unofficial].
*   **Integrated Texture:** A 3D texture storing the result of the forward integration pass, used for trilinear sampling at shaded surface fragments [S1, unofficial].
*   **FroxelPush ABI:** A push constant structure containing near/far clipping distances, tile size, number of slices, density, phase parameter, albedo, device addresses for buffers, and packed dimensions [S2, unofficial].

**Parameters (Typical Values):**
*   `DEPTH_SLICES`: 48 [S2, unofficial]
*   `DENSITY`: 0.02 [S2, unofficial]
*   `HEIGHT_FALLOFF`: 0.06 [S2, unofficial]
*   `PHASE_G`: 0.60 [S2, unofficial]
*   `SCATTER_ALBEDO`: 0.95 [S2, unofficial]
*   `NOISE_INTENSITY`: 0.15 [S2, unofficial]
*   `NOISE_SCALE`: 6.0 [S2, unofficial]
*   `INTENSITY`: 1.0 [S2, unofficial]
*   `TEMPORAL_BLEND`: 0.80 [S2, unofficial]
*   `TILE_SIZE`: 8 [S2, unofficial]

**Steps:**

1.  **Lighting Pass (Per Froxel):**
    *   One compute thread processes each froxel.
    *   Check if history can be reprojected. If reprojection fails (e.g., vector points outside history buffer), skip sub-froxel temporal jittering [S1, unofficial].
    *   If reprojection succeeds, acquire a sub-froxel jitter offset from a pre-calculated Halton(2,3,5) sequence of length 16. Apply jitter offset (z-axis only for depth-only jittering) [S1, unofficial].
    *   Evaluate participating media density using a smooth value-noise field and exponential height falloff [S2, unofficial].
    *   If the camera is submerged, boost fog density by a factor of 8 [S2, unofficial].
    *   Iterate light sources (typically dominant sun/moon). Cast 4 jittered ray-query occlusion rays toward the light source using the same sky module and transmittance LUT as the path tracer [S2, unofficial].
    *   Only froxels with non-negligible density cast shadow rays [S2, unofficial].
    *   Accumulate in-scattered radiance using shadow maps/occlusion results and attenuation. Add ambient volumetric lighting and global illumination [S1, unofficial].
    *   Evaluate local density by iterating volumetric shape nodes and combining with global fog density [S1, unofficial].
    *   Store result in the Volume Buffer: RGB = in-scatter, W = extinction [S2, unofficial].

2.  **TAA Resolve Pass:**
    *   Blend newly sampled radiance and density with the reprojected historical froxel value [S1, unofficial].
    *   Perform history validation (e.g., check for disocclusion or invalid motion vectors) [S1, unofficial].
    *   Apply temporal Exponential Moving Average (EMA) to reduce flicker and stabilize the view when the camera is still. This is gated by a still camera check and a `historyReady` guard to prevent reading uninitialized memory on the first frame or after resize [S2, unofficial].
    *   Formula: $L_{resolved} = \alpha \cdot L_{new} + (1 - \alpha) \cdot L_{history}$, where $\alpha$ is `TEMPORAL_BLEND` [S2, unofficial].

3.  **History Copy:**
    *   Copy TAA resolved data to the history buffer for the next frame. This is done in a separate pass to avoid reading and writing to the same buffer in the same pass [S1, unofficial].

4.  **Optional Filtering:**
    *   Apply a low-pass filter to TAA resolved data in the x and y directions of the froxel grid. Skipped if user disables volumetric fog filtering [S1, unofficial].

5.  **Integration Pass (Per Pixel):**
    *   One compute thread per pixel reconstructs the view ray and surface distance from the reversed-Z guide depth [S2, unofficial].
    *   March the froxel column along the view ray in front-to-back order [S1, unofficial].
    *   Calculate transmittance using the Beer-Lambert law: $T(\Delta d) = e^{-\Delta d \cdot t}$, where $\Delta d$ is the distance between consecutive depth slices and $t$ is the extinction coefficient (local participating media density) [S1, unofficial]. Note: $\Delta d$ varies due to the non-linear exponential depth distribution [S1, unofficial].
    *   Accumulate in-scattered radiance and apply transmittance.
    *   Emit result: $fogInScatter + transmittance \cdot surfaceRadiance$ [S2, unofficial].
    *   Store accumulated radiance in the Integrated Texture [S1, unofficial].
    *   If the feature is disabled, perform a 1:1 copy to ensure the frame graph has a valid image [S2, unofficial].

6.  **Composite Pass:**
    *   Blend integrated volumetric lighting with the rendered scene color [S1, unofficial].
    *   For each pixel, transform the shaded fragment position into froxel space.
    *   Trilinearly sample the Integrated Texture to retrieve the volumetric contribution [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
HALTON_SEQUENCE_LENGTH = 16
NUM_SHADOW_RAYS = 4
SUBMERGED_DENSITY_BOOST = 8.0
TEMPORAL_BLEND_FACTOR = 0.80

# Data Structures
class FroxelPushConstants:
    near: float
    far: float
    tile_size: int
    num_slices: int
    density: float
    phase_g: float
    albedo: float
    # ... other params

def halton_2_3_5(index: int) -> float:
    """Returns a value from the Halton(2,3,5) sequence."""
    # Implementation of Halton sequence generator
    pass

def hen_yeen_greenstein_phase(g: float, cos_theta: float) -> float:
    """Normalized Henyey-Greenstein phase function."""
    g_sq = g * g
    denom = 1.0 + g_sq - 2.0 * g * cos_theta
    return (1.0 - g_sq) / (4.0 * 3.14159 * denom * sqrt(denom))

def value_noise_3d(pos: vec3, scale: float, intensity: float) -> float:
    """Smooth value noise field."""
    # Implementation of 3D value noise
    pass

def exponential_depth_distribution(z_slice: int, num_slices: int, near: float, far: float) -> float:
    """Calculates the Z position for a given slice index."""
    # Exponential distribution logic
    pass

def froxel_lighting_pass(froxel_id: int, push_constants: FroxelPushConstants, history_buffer: Texture3D, current_frame: int) -> vec4:
    """
    Computes in-scattered radiance and extinction for a single froxel.
    """
    # 1. Reprojection Check
    history_valid = False
    jitter_offset = 0.0
    
    if current_frame > 0 and history_buffer.is_valid(froxel_id):
        # Check if reprojection vector is within bounds
        if is_reprojection_valid(froxel_id):
            history_valid = True
            # Acquire jitter offset
            jitter_offset = halton_2_3_5((froxel_id % HALTON_SEQUENCE_LENGTH))
        else:
            history_valid = False
            
    # 2. Density Evaluation
    base_density = push_constants.density
    height_falloff = push_constants.height_falloff
    
    # Apply height falloff
    froxel_center_y = get_froxel_y(froxel_id)
    density = base_density * exp(-height_falloff * froxel_center_y)
    
    # Add noise
    noise_val = value_noise_3d(get_froxel_center(froxel_id), push_constants.noise_scale, push_constants.noise_intensity)
    density += noise_val
    
    # Submerged boost
    if is_camera_submerged():
        density *= SUBMERGED_DENSITY_BOOST
        
    # 3. Lighting and Shadowing
    in_scatter = vec3(0.0)
    extinction = 0.0
    
    # Only cast shadows if density is non-negligible
    if density > 0.001:
        dominant_light = get_dominant_sun_moon()
        
        # Cast jittered shadow rays
        for i in range(NUM_SHADOW_RAYS):
            jitter = jitter_offset * (i + 1) # Simple jitter variation
            ray_origin = get_froxel_origin(froxel_id) + vec3(0.0, 0.0, jitter)
            ray_direction = normalize(dominant_light.position - ray_origin)
            
            # Ray query for occlusion
            occlusion = ray_query_occlusion(ray_origin, ray_direction, dominant_light)
            
            # Accumulate radiance
            phase_val = hen_yeen_greenstein_phase(push_constants.phase_g, dot(ray_direction, normalize(dominant_light.position)))
            light_intensity = dominant_light.intensity * occlusion
            in_scatter += light_intensity * push_constants.albedo * phase_val
            
        # Add ambient and GI
        in_scatter += get_ambient_volumetric_lighting() * push_constants.albedo
        in_scatter += get_global_illumination_contribution() * push_constants.albedo
        
    extinction = density * push_constants.intensity
    
    return vec4(in_scatter, extinction)

def taa_resolve_pass(froxel_id: int, current_radiance: vec4, history_buffer: Texture3D, is_still_camera: bool) -> vec4:
    """
    Blends current frame with history using EMA.
    """
    if is_still_camera and history_buffer.is_valid(froxel_id):
        history_radiance = history_buffer.sample(froxel_id)
        # EMA Blend
        resolved_radiance = TEMPORAL_BLEND_FACTOR * current_radiance.rgb + (1.0 - TEMPORAL_BLEND_FACTOR) * history_radiance.rgb
        resolved_density = TEMPORAL_BLEND_FACTOR * current_radiance.a + (1.0 - TEMPORAL_BLEND_FACTOR) * history_radiance.a
        return vec4(resolved_radiance, resolved_density)
    else:
        return current_radiance

def integration_pass(pixel_id: int, view_ray: vec3, surface_dist: float, volume_buffer: Texture3D, integrated_texture: Texture3D) -> vec4:
    """
    Marches froxel column along view ray.
    """
    accumulated_radiance = vec3(0.0)
    transmittance = 1.0
    
    # Reconstruct froxel indices along view ray
    froxel_indices = get_froxel_indices_along_ray(view_ray, surface_dist)
    
    for i in range(len(froxel_indices) - 1):
        curr_idx = froxel_indices[i]
        next_idx = froxel_indices[i + 1]
        
        # Get froxel data
        froxel_data = volume_buffer.sample(curr_idx)
        radiance = froxel_data.rgb
        density = froxel_data.a
        
        # Calculate distance between slices
        delta_d = get_distance_between_slices(curr_idx, next_idx)
        
        # Beer-Lambert Law
        extinction_coeff = density
        t = exp(-delta_d * extinction_coeff)
        
        # Accumulate
        accumulated_radiance += radiance * transmittance * (1.0 - t) / extinction_coeff if extinction_coeff > 0 else vec3(0.0)
        transmittance *= t
        
    # Final composite with surface radiance
    surface_radiance = get_surface_radiance(pixel_id)
    final_radiance = accumulated_radiance + transmittance * surface_radiance
    
    return vec4(final_radiance, transmittance)

def composite_pass(fragment_pos: vec3, integrated_texture: Texture3D, scene_color: vec3) -> vec3:
    """
    Blends volumetric lighting with scene color.
    """
    # Transform to froxel space
    froxel_pos = transform_to_froxel_space(fragment_pos)
    
    # Trilinear sampling
    vol_data = integrated_texture.trilinear_sample(froxel_pos)
    vol_radiance = vol_data.rgb
    vol_transmittance = vol_data.a
    
    # Blend
    final_color = vol_radiance + scene_color * vol_transmittance
    return final_color
```

## Subtopics

### [Froxel TAA Reprojection](1920-froxel-taa-reprojection.md)
A temporal anti-aliasing technique for froxel-based volumetric rendering that achieves temporal supersampling by reusing in-scattered radiance data from the previous frame and blending it with new stochastic samples. It solves the heavy aliasing inherent in single-sample-per-froxel rendering by estimating the light transport integral over time using an Exponential Moving Average (EMA). Unlike traditional pixel-based TAA which relies on geometry and depth buffers to detect disocclusions, this method operates on world-space volumetric data, requiring only camera motion reprojection and handling history validity through frustum bounds and neighborhood color extents.

### [Froxel Depth Distribution](1922-froxel-depth-distribution.md)
A non-linear depth distribution for froxel grids that makes nearby froxels smaller to capture volumetric detail where needed most.

### [Volumetric Accumulation Rendering Pipeline](1932-volumetric-accumulation-rendering-pipeline.md)
The Volumetric Accumulation Rendering Pipeline is a real-time approximation method for light transport in participating media that uses a low-resolution froxel grid to compute in-scattered radiance and applies temporal anti-aliasing (TAA) to stabilize the result. It solves the high computational cost and memory requirements of full volumetric ray-marching by decoupling the expensive scattering calculation from the final high-resolution rendering pass, relying on history reprojecting and exponential smoothing to reduce noise. It differs from deterministic ray-marching compositing by accepting stochastic noise in exchange for performance, and from static froxel shading by introducing temporal coherence to handle camera movement and dynamic scenes.

### [Froxel Volumetric Fog with Path-Traced Shadows](2314-froxel-volumetric-fog-with-path-traced-shadows.md)
Froxel Volumetric Fog with Path-Traced Shadows is a real-time rendering technique that simulates light scattering in participating media by dividing the camera view frustum into a 3D grid of voxels ("froxels") and computing their illumination via path-traced Next Event Estimation (NEE) shadow rays. Unlike sibling methods that rely on shadow maps or precomputed lighting passes, this approach inherits real color transmission through tinted dielectrics (e.g., magenta glass scattering magenta fog) and dynamic sky-dome lighting, distinguishing it from [Froxel Volumetric Fog with Precomputed Lighting](2647-froxel-volumetric-fog-with-precomputed-lighting.md) which uses compute-shader approximations. It solves the problem of flat, uniform screen-space fog by providing spatially varying, shadowed, and spectrally accurate volumetric lighting at the cost of higher computational complexity per voxel.

### [Fog Volume Grid Rendering](4350-fog-volume-grid-rendering.md)
A technique to draw FogVolume and FogMaterial nodes into the volumetric fog by adding their density, albedo, and emission to the froxels (volumetric grid cells) that the volume's shape covers, matching Godot 4.6.3's behavior.

## Operating characteristics
*   **Time Complexity:** Lighting work is amortized to once per froxel instead of per-pixel per step, delivering spatial-accumulation savings [S2, unofficial]. The integration pass is O(N) per pixel, where N is the number of froxel slices along the view ray.
*   **Memory Complexity:** Requires storage for the Volume Buffer (16-bit RGBA 3D texture), History Buffer, and Integrated Texture. Size depends on tile resolution and depth slice count (e.g., 48 slices) [S2, unofficial].
*   **Performance:** Delivers Minecraft-RTX-like atmospheric fog and dynamic sun/moon god rays/light shafts at real-time frame rates [S2, unofficial]. Feeds into DLSS-RR (Deep Learning Super Sampling with Reconstruction) [S2, unofficial].
*   **Strengths:** Amortizes expensive ray-query occlusion work; handles dynamic lighting and shadows; supports temporal anti-aliasing for noise reduction.
*   **Limitations:** Requires RayQuery support in compute shaders (which may have compilation issues in some environments); push constant size must fit within device limits; temporal artifacts may occur if history validation fails [S2, unofficial].
*   **When to use:** Real-time applications requiring dynamic volumetric fog, light shafts, and atmospheric scattering with shadowing.
*   **When not to use:** Applications requiring precise spectral scattering through complex dielectrics without path-traced shadows, or where hardware does not support RayQuery in compute shaders.

## Background
Froxel-based Volumetric Rendering is based on the Wronski/Frostbite-style frustum-aligned volumetric fog technique [S2, unofficial]. It was implemented in the LumisCaustica project by Lumitos0-0 and others, with commits added in August 2026 [S2, unofficial]. The method is also featured in Godot's volumetric rendering pipeline, as described in research on temporal anti-aliasing for real-time froxel-based volumetric rendering [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time rendering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S2] Froxel-based volumetric fog and light shafts, https://github.com/Lumitos0-0/LumisCaustica/pull/21, fetched 2026-10-26 (unofficial)
