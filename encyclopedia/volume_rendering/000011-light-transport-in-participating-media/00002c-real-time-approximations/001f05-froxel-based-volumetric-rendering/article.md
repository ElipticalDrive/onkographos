# Froxel-based Volumetric Rendering

Also called: Froxel Rendering, Froxel-based volumetric fog and light shafts, Godot Volumetric Rendering Pipeline, Wronski/Frostbite-style frustum-aligned volumetric fog, froxels

## Overview
Froxel-based Volumetric Rendering is a real-time approximation for light transport in participating media that amortizes ray-queried occlusion lighting work by dividing the camera view frustum into a 3D grid of voxels ("froxels") [S2, unofficial]. It solves the high computational cost of per-pixel ray marching by collapsing per-pixel fog cost to a fixed march in an integrate pass, delivering spatial-accumulation savings [S2, unofficial]. Unlike [Voxel Cone Tracing](1454-voxel-cone-tracing.md) which relies on cone tracing for indirect lighting, or [Precomputed Lighting (Baking)](2169-precomputed-lighting-baking.md) which uses static data, this method computes dynamic in-scattered radiance and shadows per froxel, often combined with [Froxel TAA Reprojection](1920-froxel-taa-reprojection.md) to stabilize temporal noise [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera frustum parameters (near/far planes), tile size, depth slice count, fog density parameters, phase function parameters, albedo, noise parameters, and scene geometry (via TLAS for ray queries) [S2, unofficial].
*   **Outputs:** A 3D texture containing integrated volumetric radiance and density, which is sampled during the final composite pass to blend fog with the scene [S1, unofficial].

**Data Structures:**
*   **Froxel Grid:** A frustum-aligned 3D grid where the XY dimensions correspond to a screen tile footprint and the Z dimension uses an exponential depth distribution [S2, unofficial].
*   **Volume Buffer:** A 16-bit RGBA 3D texture where RGB channels store wavelength-dependent in-scattered radiance and the Alpha/W channel stores participating media density or extinction [S1, unofficial][S2, unofficial].
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
    *   One compute thread processes each froxel [S2, unofficial].
    *   Check if history can be reprojected. If reprojection fails (e.g., vector points outside history buffer), skip sub-froxel temporal jittering [S1, unofficial].
    *   If reprojection succeeds, acquire a sub-froxel jitter offset from a pre-calculated Halton(2,3,5) sequence of length 16 [S1, unofficial].
    *   Apply jitter offset (z-axis only for depth-only jittering) [S1, unofficial].
    *   Evaluate participating media density using a smooth value-noise field and exponential height falloff [S2, unofficial].
    *   If the camera is submerged, boost fog density by a factor of 8 [S2, unofficial].
    *   Iterate light sources (typically dominant sun/moon) [S2, unofficial].
    *   Cast 4 jittered ray-query occlusion rays toward the light source using the same sky module and transmittance LUT as the path tracer [S2, unofficial].
    *   Only froxels with non-negligible density cast shadow rays [S2, unofficial].
    *   Accumulate in-scattered radiance using shadow maps/occlusion results and attenuation [S1, unofficial].
    *   Add ambient volumetric lighting and global illumination [S1, unofficial].
    *   Evaluate local density by iterating volumetric shape nodes and combining with global fog density [S1, unofficial].
    *   Store result in the Volume Buffer: RGB = in-scatter, W = extinction [S2, unofficial].

2.  **TAA Resolve Pass:**
    *   Blend newly sampled radiance and density with the reprojected historical froxel value [S1, unofficial].
    *   Perform history validation (e.g., check for disocclusion or invalid motion vectors) [S1, unofficial].
    *   Apply temporal Exponential Moving Average (EMA) to reduce flicker and stabilize the view when the camera is still [S2, unofficial].
    *   Formula: $L_{resolved} = \alpha \cdot L_{new} + (1 - \alpha) \cdot L_{history}$, where $\alpha$ is `TEMPORAL_BLEND` [S2, unofficial].

3.  **History Copy:**
    *   Copy TAA resolved data to the history buffer for the next frame [S1, unofficial].
    *   This is done in a separate pass to avoid reading and writing to the same buffer in the same pass [S1, unofficial].

4.  **Optional Filtering:**
    *   Apply a low-pass filter to TAA resolved data in the x and y directions of the froxel grid [S1, unofficial].
    *   Skipped if user disables volumetric fog filtering [S1, unofficial].

5.  **Integration Pass (Per Pixel):**
    *   One compute thread per pixel reconstructs the view ray and surface distance from the reversed-Z guide depth [S2, unofficial].
    *   March the froxel column along the view ray in front-to-back order [S1, unofficial].
    *   Calculate transmittance using the Beer-Lambert law: $T(\Delta d) = e^{-\Delta d \cdot t}$, where $\Delta d$ is the distance between consecutive depth slices and $t$ is the extinction coefficient (local participating media density) [S1, unofficial].
    *   Note: $\Delta d$ varies due to the non-linear exponential depth distribution [S1, unofficial].
    *   Accumulate in-scattered radiance and apply transmittance [S1, unofficial].
    *   Emit result: $fogInScatter + transmittance \cdot surfaceRadiance$ [S2, unofficial].
    *   Store accumulated radiance in the Integrated Texture [S1, unofficial].
    *   If the feature is disabled, perform a 1:1 copy to ensure the frame graph has a valid image [S2, unofficial].

6.  **Composite Pass:**
    *   Blend integrated volumetric lighting with the rendered scene color [S1, unofficial].
    *   For each pixel, transform the shaded fragment position into froxel space [S1, unofficial].
    *   Trilinearly sample the Integrated Texture to retrieve the volumetric contribution [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants from evidence
HALTON_SEQUENCE_LENGTH = 16
NUM_SHADOW_RAYS = 4
SUBMERGED_DENSITY_BOOST = 8.0
TEMPORAL_BLEND_FACTOR = 0.80

def halton_sequence(base: int, index: int) -> float:
    """Standard Halton sequence generator [G]."""
    result = 0.0
    f = 1.0
    i = index
    while i > 0:
        f = f / base
        result = result + f * (i % base)
        i = i // base
    return result

def hen_yey_greenstein_phase(g: float, cos_theta: float) -> float:
    """Normalized Henyey-Greenstein phase function [G]."""
    g_sq = g * g
    denom = 1.0 + g_sq - 2.0 * g * cos_theta
    return (1.0 - g_sq) / (4.0 * math.pi * math.pow(denom, 1.5))

def value_noise_3d(x: float, y: float, z: float, scale: float, intensity: float) -> float:
    """Deterministic 3D value noise for fog field [G]."""
    p = (x * scale, y * scale, z * scale)
    h = int(p[0] * 127.1 + p[1] * 311.7 + p[2] * 74.7) % 1000
    return (h / 1000.0 - 0.5) * intensity

def exponential_depth(z_idx: int, num_slices: int, near: float, far: float) -> float:
    """Exponential depth distribution for Z slices [G]."""
    if num_slices <= 1:
        return far
    t = z_idx / (num_slices - 1)
    return near * math.pow(far / near, t)

def froxel_lighting(froxel_id: int, tile_size: int, num_slices: int, near: float, far: float,
                    density: float, height_falloff: float, phase_g: float, albedo: float,
                    noise_scale: float, noise_intensity: float, intensity: float,
                    camera_submerged: bool, light_dir: tuple, history_valid: bool,
                    history_ready: bool) -> tuple:
    """Pass 1: Lighting pass. One compute thread per froxel."""
    # 1. Reprojection & Jitter
    jitter_offset = 0.0
    if history_valid and history_ready:
        jitter_offset = halton_sequence(2, froxel_id % HALTON_SEQUENCE_LENGTH)

    # 2. Density Evaluation
    x_idx = froxel_id // (tile_size * num_slices)
    y_idx = (froxel_id // num_slices) % tile_size
    z_idx = froxel_id % num_slices
    froxel_center = (x_idx, y_idx, z_idx)
    froxel_y = froxel_center[1]

    base_density = density * math.exp(-height_falloff * froxel_y)
    noise_val = value_noise_3d(froxel_center[0], froxel_center[1], froxel_center[2], noise_scale, noise_intensity)
    local_density = base_density + noise_val

    if camera_submerged:
        local_density *= SUBMERGED_DENSITY_BOOST

    # 3. Lighting & Shadowing
    in_scatter = (0.0, 0.0, 0.0)
    if local_density > 0.001:
        for i in range(NUM_SHADOW_RAYS):
            ray_origin = (froxel_center[0], froxel_center[1], froxel_center[2] + jitter_offset * (i + 1))
            ray_direction = normalize(light_dir, ray_origin)
            occlusion = ray_query_occlusion(ray_origin, ray_direction, far)
            phase_val = hen_yey_greenstein_phase(phase_g, max(0.0, dot(ray_direction, normalize(light_dir))))
            in_scatter = (in_scatter[0] + occlusion * albedo * phase_val,
                          in_scatter[1] + occlusion * albedo * phase_val,
                          in_scatter[2] + occlusion * albedo * phase_val)
        in_scatter = (in_scatter[0] + albedo * 0.1, in_scatter[1] + albedo * 0.1, in_scatter[2] + albedo * 0.1)
        in_scatter = (in_scatter[0] + albedo * 0.05, in_scatter[1] + albedo * 0.05, in_scatter[2] + albedo * 0.05)

    extinction = local_density * intensity
    return (in_scatter, extinction)

def taa_resolve(current_radiance: tuple, history_radiance: tuple, is_still_camera: bool, history_ready: bool) -> tuple:
    """Pass 2: TAA Resolve. Blend new with history."""
    if is_still_camera and history_ready:
        resolved_rgb = (TEMPORAL_BLEND_FACTOR * current_radiance[0][i] + (1.0 - TEMPORAL_BLEND_FACTOR) * history_radiance[0][i] for i in range(3))
        resolved_a = TEMPORAL_BLEND_FACTOR * current_radiance[1] + (1.0 - TEMPORAL_BLEND_FACTOR) * history_radiance[1]
        return (tuple(resolved_rgb), resolved_a)
    return current_radiance

def integration_pass(view_ray: tuple, surface_dist: float, volume_buffer: list, num_slices: int, near: float, far: float) -> tuple:
    """Pass 5: Integration pass. One compute thread per pixel."""
    accumulated_radiance = (0.0, 0.0, 0.0)
    transmittance = 1.0
    froxel_indices = list(range(num_slices))

    for i in range(len(froxel_indices) - 1):
        curr_idx = froxel_indices[i]
        next_idx = froxel_indices[i + 1]
        froxel_data = volume_buffer[curr_idx]
        radiance = froxel_data[0]
        density = froxel_data[1]

        curr_z = exponential_depth(curr_idx, num_slices, near, far)
        next_z = exponential_depth(next_idx, num_slices, near, far)
        delta_d = abs(next_z - curr_z)

        extinction_coeff = density
        t = math.exp(-delta_d * extinction_coeff)

        if extinction_coeff > 0:
            accumulated_radiance = (accumulated_radiance[0] + radiance[0] * transmittance * (1.0 - t) / extinction_coeff,
                                    accumulated_radiance[1] + radiance[1] * transmittance * (1.0 - t) / extinction_coeff,
                                    accumulated_radiance[2] + radiance[2] * transmittance * (1.0 - t) / extinction_coeff)
        transmittance *= t

    final_radiance = (accumulated_radiance[0] + transmittance * surface_dist,
                      accumulated_radiance[1] + transmittance * surface_dist,
                      accumulated_radiance[2] + transmittance * surface_dist)
    return (final_radiance, transmittance)

def composite_pass(fragment_pos: tuple, integrated_texture: list, scene_color: tuple) -> tuple:
    """Pass 6: Composite pass."""
    froxel_pos = fragment_pos
    vol_data = integrated_texture[froxel_pos]
    vol_radiance = vol_data[0]
    vol_transmittance = vol_data[1]
    return (vol_radiance[0] + scene_color[0] * vol_transmittance,
            vol_radiance[1] + scene_color[1] * vol_transmittance,
            vol_radiance[2] + scene_color[2] * vol_transmittance)

def normalize(light_dir: tuple, ray_origin: tuple) -> tuple:
    """Standard vector normalization [G]."""
    dx = light_dir[0] - ray_origin[0]
    dy = light_dir[1] - ray_origin[1]
    dz = light_dir[2] - ray_origin[2]
    length = math.sqrt(dx*dx + dy*dy + dz*dz)
    return (dx/length, dy/length, dz/length)

def dot(v1: tuple, v2: tuple) -> float:
    """Standard dot product [G]."""
    return v1[0]*v2[0] + v1[1]*v2[1] + v1[2]*v2[2]

def ray_query_occlusion(origin: tuple, direction: tuple, max_dist: float) -> float:
    """Hardware ray-query or step-based occlusion check [G]."""
    # Simplified step-based occlusion for self-containment
    step = 0.1
    t = 0.0
    while t < max_dist:
        pos = (origin[0] + direction[0]*t, origin[1] + direction[1]*t, origin[2] + direction[2]*t)
        if is_occluded(pos):
            return 0.0
        t += step
    return 1.0

def is_occluded(pos: tuple) -> bool:
    """Placeholder for geometry intersection test [G]."""
    return False
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
*   **Time Complexity:** Lighting work is amortized to once per froxel instead of per-pixel per step, delivering spatial-accumulation savings [S2, unofficial]. The integration pass is O(N) per pixel, where N is the number of froxel slices along the view ray [S1, unofficial].
*   **Memory Complexity:** Requires storage for the Volume Buffer (16-bit RGBA 3D texture), History Buffer, and Integrated Texture [S1, unofficial]. Size depends on tile resolution and depth slice count (e.g., 48 slices) [S2, unofficial].
*   **Performance:** Delivers Minecraft-RTX-like atmospheric fog and dynamic sun/moon god rays/light shafts at real-time frame rates [S2, unofficial]. Feeds into DLSS-RR (Deep Learning Super Sampling with Reconstruction) [S2, unofficial].
*   **Strengths:** Amortizes expensive ray-query occlusion work; handles dynamic lighting and shadows; supports temporal anti-aliasing for noise reduction [S2, unofficial].
*   **Limitations:** Requires RayQuery support in compute shaders (which may have compilation issues in some environments); push constant size must fit within device limits; temporal artifacts may occur if history validation fails [S2, unofficial].
*   **When to use:** Real-time applications requiring dynamic volumetric fog, light shafts, and atmospheric scattering with shadowing [S2, unofficial].
*   **When not to use:** Applications requiring precise spectral scattering through complex dielectrics without path-traced shadows, or where hardware does not support RayQuery in compute shaders [S2, unofficial].

## Background
Froxel-based Volumetric Rendering is based on the Wronski/Frostbite-style frustum-aligned volumetric fog technique [S2, unofficial]. It was implemented in the LumisCaustica project by Lumitos0-0 and others, with commits added in August 2026 [S2, unofficial]. The method is also featured in Godot's volumetric rendering pipeline, as described in research on temporal anti-aliasing for real-time froxel-based volumetric rendering [S1, unofficial].

## Sources
[S1] Temporal Anti-Aliasing for Real-Time Froxel-Based Volumetric Rendering - Advances in volumetric temporal anti-aliasing for real-time rendering of participating media, https://openalex.org/W7218067689, fetched 2026-11-01 (unofficial)
[S2] Froxel-based volumetric fog and light shafts, https://github.com/Lumitos0-0/LumisCaustica/pull/21, fetched 2026-10-26 (unofficial)
