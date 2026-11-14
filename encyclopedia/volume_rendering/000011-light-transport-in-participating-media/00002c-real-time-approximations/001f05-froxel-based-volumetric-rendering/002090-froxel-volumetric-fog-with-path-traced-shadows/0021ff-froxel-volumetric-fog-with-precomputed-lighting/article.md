# Froxel Volumetric Fog with Precomputed Lighting

Also called: Froxel Volumetric Fog, Froxel Fog, Frustum Voxel Fog

## Overview
Froxel Volumetric Fog with Precomputed Lighting is a real-time rendering technique that simulates light scattering in participating media by precomputing in-scattered radiance and optical depth in a camera-aligned 3D grid (froxels) [S3, S4, unofficial]. It solves the performance bottleneck of full ray-marched volumetric lighting by decoupling the expensive light transport calculation (performed once per frame in a compute shader) from the per-pixel integration (a simple front-to-back accumulation) [S3, unofficial]. Unlike its sibling [Froxel Volumetric Fog with Path-Traced Shadows](2314-froxel-volumetric-fog-with-path-traced-shadows.md), which uses Next Event Estimation for shadows, this variant typically relies on shadow maps or atlas-based occlusion for the precomputation step, offering a balance between visual fidelity and frame budget [S2, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Camera parameters: Near/Far planes, View/Projection matrices, Screen resolution.
*   Scene data: Opaque depth buffer, Shadow maps (or light atlas), Light positions/colors, Sky ambient.
*   Fog parameters: Base density, height falloff, anisotropy ($g$), max distance, noise texture.
*   History: Previous frame's froxel volume (for temporal reprojection).

**Outputs:**
*   Integrated froxel volume: A 3D texture storing accumulated in-scattered radiance (RGB) and remaining transmittance (Alpha) [S3, unofficial].
*   Final image: Scene color composited with the integrated fog volume.

### Data Structures

**1. Froxel Volume (Scattering/Extinction)**
*   **Format:** `RGBA16F` (or `R16G16B16A16_SFloat`) [S2, S4, unofficial].
*   **Dimensions:** Variable based on quality preset.
    *   Low: $80 \times 45 \times 64$
    *   Medium: $160 \times 90 \times 64$
    *   High: $160 \times 90 \times 128$
    *   Ultra: $240 \times 135 \times 128$ [S3, unofficial]
*   **Layout:**
    *   X, Y: Screen tiles (aligned with screen resolution, typically sub-sampled).
    *   Z: Depth slices (exponential spacing) [S3, unofficial].
    *   RGB: In-scattered radiance (lighting) [S3, S4, unofficial].
    *   A: Extinction coefficient (density) [S3, unofficial].

**2. Integrated Volume**
*   **Format:** `RGBA16F` [S3, unofficial].
*   **Layout:**
    *   RGB: Accumulated in-scattered radiance after front-to-back integration [S3, unofficial].
    *   A: Remaining transmittance after integration [S3, unofficial].

**3. Frame Constants**
*   `volumetric_fog_control`: A constant block appended to `FrameViewData`/`CyFrameData` at offset 528. Total block size is 528 bytes (or 544 bytes with decal control) [S1, S2, unofficial].

### Numbered Steps

1.  **Noise Bake:** Sample a prebaked 128×128×128 tileable 3D noise texture to drive weather-driven drift [S3, unofficial].
2.  **Inject Density:** Compute the extinction coefficient for each froxel based on height falloff and noise.
3.  **Scatter Lighting:** Compute the in-scattered radiance for each froxel using precomputed lighting (sun, ambient, local lights) and shadow maps [S3, unofficial].
4.  **Temporal Reprojection:** Blend the current scattering volume with the previous frame's history to reduce noise [S3, unofficial].
5.  **Spatial Filter:** Apply a 3×3×3 Gaussian filter to the RGB scattering data only (extinction passes through) [S3, unofficial].
6.  **Integrate:** Perform a front-to-back march along the Z-axis for each screen-space XY cell to compute optical depth and accumulated scatter [S3, unofficial].
7.  **Composite:** Sample the integrated volume using screen UVs and scene depth, then blend with the scene color [S3, unofficial].
8.  **Update Opaque Texture:** Copy the fogged color into `_CameraOpaqueTexture` so refractive materials (glass, windshields) see the fogged world [S3, unofficial].

### Formulas

**Depth Slice Position (Exponential):**
$$ depth = nearDistance \times \exp\left(\frac{slice}{sliceCount} \times \log\left(\frac{farDistance}{nearDistance}\right)\right) $$
This provides higher resolution near the camera where artifacts are most visible [S3, unofficial].

**Height Falloff Density:**
$$ h = worldPos.y - groundHeight $$
$$ density = baseDensity \times \exp\left(-\frac{\max(h, 0.0)}{\max(heightFalloff, 0.01)}\right) $$
Creates a ground-hugging fog shape [S3, unofficial].

**Animated Noise Lookup:**
$$ uvw = \frac{worldPos}{noiseScale} - wind \times \frac{time}{noiseScale} $$
$$ n = \text{noiseTexture}.\text{SampleLevel}(\text{sampler}, \text{frac}(uvw), 0) $$
Provides drift without procedural noise evaluation [S3, unofficial].

**Henyey-Greenstein Phase Function:**
$$ P(\theta) = \frac{1 - g^2}{(1 + g^2 - 2g \cos \theta)^{3/2}} $$
Where $g$ is the anisotropy parameter ($|g| < 1$). $g \approx 0$ scatters softly; positive $g$ pushes energy forward (sun halos) [S3, unofficial].

**Integration Slices:**
$$ opticalDepth = extinction \times sliceThickness $$
$$ sliceTransmittance = \exp(-opticalDepth) $$
$$ sliceScatter = scatter \times \frac{1 - sliceTransmittance}{extinction} $$
[ S3 ]

**Compositing:**
$$ finalColor = sceneColor \times transmittance + accumulatedScatter $$
[ S3 ]

**Temporal Reprojection:**
$$ blended = \text{lerp}(history.rgb, current.rgb, temporalBlend) $$
[ S3 ]

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 2: Inject Density (Compute Shader)
def inject_density(fog_volume: RWTexture3D, world_pos: float3, params: FogParams):
    # Calculate height factor for ground-hugging fog
    h = world_pos.y - params.ground_height
    height_factor = saturate(1.0 - (h / params.fog_height))
    
    # Apply base density and height falloff
    density = params.base_density * height_factor
    
    # Optional: Add animated noise
    uvw = (world_pos / params.noise_scale) - (params.wind * (params.time / params.noise_scale))
    noise_val = sample_noise_texture(uvw)
    density *= (1.0 + noise_val * params.noise_strength)
    
    # Store extinction in Alpha, scattering in RGB (initially 0)
    fog_volume[id] = float4(0.0, 0.0, 0.0, density)

# Step 3: Scatter Lighting (Compute Shader)
def scatter_lighting(fog_volume: RWTexture3D, world_pos: float3, light: LightData):
    # Read current density/extinction
    current = fog_volume[id]
    density = current.a
    
    # Calculate scattering contribution
    # Phase function approximation (Henyey-Greenstein)
    # Note: In full implementation, use HG formula with g parameter
    scatter = saturate(dot(normalize(float3(0, 1, 0)), -light.direction))
    
    # Apply shadowing (from shadow map or atlas)
    shadow = sample_shadow_map(world_pos, light.shadow_map)
    
    # Calculate lighting
    lighting = light.color * density * scatter * shadow
    
    # Accumulate into RGB
    # Clamp to prevent temporal instability spikes from bright lights
    fog_volume[id].rgb += clamp(lighting, 0.0, params.max_light_contribution)

# Step 6: Integrate (Compute Shader or Fragment Shader)
def integrate_froxel(xy: int2, froxel_volume: Texture3D, integrated_volume: RWTexture3D):
    # March through Z (depth slices)
    accumulated_scatter = float3(0.0, 0.0, 0.0)
    transmittance = 1.0
    
    for slice in range(froxel_volume.depth):
        # Get froxel data at this slice
        froxel = froxel_volume[xy, slice]
        extinction = froxel.a
        scatter = froxel.rgb
        
        # Compute slice optical depth
        # sliceThickness depends on exponential spacing
        slice_thickness = get_slice_thickness(slice)
        optical_depth = extinction * slice_thickness
        
        # Compute slice transmittance
        slice_transmittance = exp(-optical_depth)
        
        # Compute slice scatter contribution
        # Avoid division by zero if extinction is 0
        if extinction > 0.0001:
            slice_scatter = scatter * (1.0 - slice_transmittance) / extinction
        else:
            slice_scatter = scatter * slice_thickness # Linear approx for low density
            
        # Accumulate
        accumulated_scatter += slice_scatter * transmittance
        transmittance *= slice_transmittance
        
    # Store results
    integrated_volume[xy] = float4(accumulated_scatter, transmittance)

# Step 7: Composite (Fragment Shader)
def composite_froxel(screen_uv: float2, scene_color: float4, integrated_volume: Texture3D):
    # Convert screen UV and scene depth to froxel coordinates
    froxel_z = depth_to_froxel_z(scene_color.a)
    
    # Sample integrated volume
    integrated = integrated_volume[screen_uv, froxel_z]
    accumulated_scatter = integrated.rgb
    transmittance = integrated.a
    
    # Blend with scene color
    final_color = scene_color.rgb * transmittance + accumulated_scatter
    
    return float4(final_color, scene_color.a)
```

### Parameters and Typical Values
*   **Anisotropy ($g$):** Default 0.3. Must be validated to $|g| < 1$ [S2, unofficial].
*   **Max Distance:** Typically 200–500 meters. Beyond this, analytic fog is used [S2, unofficial].
*   **Resolution:** Often 1/4 of screen resolution (e.g., 160×90 for 1280×720) [S4, unofficial].
*   **Temporal Blend Factor:** 0.9 (90% history, 10% current) [S4, unofficial].

### Edge Cases and Pitfalls
*   **Shadow Map Inside-Solid Lighting:** Shadow maps cull front faces, potentially lighting the interior of occluders. Froxels inside solids must be excluded using geometry predicates [S1, unofficial].
*   **Seam at Max Distance:** A visible ring can appear at the boundary between the froxel volume and analytic fog. Mitigated by blending or extending the volume [S2, unofficial].
*   **Banding:** High anisotropy can cause banding near forward directions. Low-frequency anisotropy noise should be kept small [S3, unofficial].
*   **Zero Normal Invariant:** Froxels use a normal of `vec3(0.0)` to prevent `normal_bias` displacement artifacts [S2, unofficial].
*   **Temporal Instability:** Bright point/spot lights near their source can cause spikes. Light contributions must be clamped before entering the volume [S3, unofficial].

## Subtopics

### [Froxel Fog with Temporal Reprojection and Revealed Froxel Blur](13212-froxel-fog-with-temporal-reprojection-and-revealed-froxel-bl.md)
This variant focuses on the temporal stability and spatial smoothing of the froxel volume. It uses temporal reprojection to blend the current frame with the previous frame using a lerp factor (e.g., 0.9) to reduce noise and stabilize lighting. It also employs a revealed froxel blur, which is a 3x3x3 Gaussian-style filter applied to the scattering volume after reprojection but before history storage to prevent blur stacking. This filter applies only to RGB (scattering), leaving extinction (alpha) unchanged to preserve depth and absorption profiles.

### [Volumetric Particle Injection](13347-volumetric-particle-injection.md)
This variant extends the froxel grid to include dynamic volumetric particles (smoke, steam, energy discharges). Particles are injected as density into the frustum-aligned froxel grid, allowing them to be lit by the same scattering pass as the global fog. This avoids visible seams that would occur if particles were lit by a decoupled lighting model.

## Operating characteristics

**Time Complexity:**
*   **Scattering Pass:** $O(N_{froxels} \times N_{lights})$. The cost depends on the number of froxels and light sources. In practice, only dominant lights (sun, main directional) are used for the bulk of the scattering, with local lights added as secondary contributions [S3, S5, unofficial].
*   **Integration Pass:** $O(N_{screen\_pixels} \times N_{slices})$. Linear with screen resolution and depth slices.
*   **Measured Performance:** On an NVIDIA RTX 4090 at 2560×1440, the FOG_SCATTERING + FOG_INTEGRATION passes take approximately 0.043 ms (forward) and 0.045 ms (deferred) [S2, unofficial]. This is ~10× faster than the 0.5 ms target for 160×90×64 resolution with a four-cascade sun [S2, unofficial].

**Memory Complexity:**
*   **Volume Storage:** $O(W \times H \times D \times 4 \text{ bytes})$. For a 160×90×64 RGBA16F volume, this is approximately 7.03 MiB per view [S2, unofficial].
*   **History Storage:** Two ping-ponged 3D render textures for temporal reprojection [S3, unofficial].

**Strengths:**
*   Real-time performance suitable for AAA games [S4, unofficial].
*   Reacts to dynamic lights, shadows, and weather [S3, unofficial].
*   Supports refractive materials seeing the fog [S3, unofficial].
*   Exponential depth slices optimize resolution where it matters most [S3, unofficial].

**Limitations:**
*   Limited to single scattering (or simple octave approximations) [S5, unofficial].
*   Cannot handle complex global illumination or multiple scattering accurately [S5, unofficial].
*   Performance scales with light count and froxel resolution [S5, unofficial].
*   Potential for banding and seams at max distance [S2, unofficial].

**When to Use:**
*   Real-time applications requiring atmospheric fog with dynamic lighting.
*   Scenes with significant height variation (ground-hugging fog).
*   When refractive materials need to interact with the fog.

**When Not to Use:**
*   Offline rendering requiring high-fidelity multiple scattering.
*   Scenes with extremely complex, dynamic volumetric geometry that exceeds the froxel grid's resolution.
*   Mobile devices with very limited memory and compute power (unless using very low quality presets).

## Background

**Origin:**
The term "froxel" (frustum voxel) is attributed to Alex Evans and Sony ATG, described in the "Learning from Failure" SIGGRAPH 2015 talk and noted in the Real-Time Rendering blog and Call of Duty: Advanced Warfare post-processing [S3, unofficial]. The technique was further developed in works such as "Volumetric Fog: Unified, Compute Shader Based Solution to Atmospheric Scattering" by Bartlomiej Wronski (SIGGRAPH 2014) and "Physically-based and Unified Volumetric Rendering in Frostbite" by Sebastien Hillaire (SIGGRAPH 2015) [S3, unofficial].

**History:**
*   **1941:** Henyey and Greenstein publish the phase function used for scattering [S3, unofficial].
*   **2014:** Wronski presents a compute shader-based solution for atmospheric scattering [S3, unofficial].
*   **2015:** Alex Evans introduces the "froxel" concept [S3, unofficial].
*   **2026:** Implementation in Cyberdyne Game Engine (PR #39) and c3d.c3l project (PR #186) [S1, S2, unofficial].

**Usage:**
Successfully used in commercial games like *Assassin's Creed IV* (Wronski's method) [S5, unofficial]. Implemented in the Cyberdyne Game Engine and c3d.c3l project for sky, terrain, vegetation, and water examples [S1, S2, unofficial].

## Sources

[S1] Add volumetric fog: froxel medium, shadowed march and frame stage, https://github.com/CyberdyneCorp/CyberdyneEngine/pull/39, fetched 2026-10-26 (unofficial)
[S2] feat(render): froxel volumetric fog, https://github.com/fesoliveira014/c3d.c3l/pull/186, fetched 2026-10-26 (unofficial)
[S3] Building a froxel volumetric fog renderer in Unity URP, https://www.euclideandreams.com/writing/building-a-froxel-volumetric-fog-renderer-in-unity-urp, fetched 2026-10-27 (unofficial)
[S4] Real‑Time Volumetric Fog in Unity Using 3D Textures and Compute Shaders August 24, 2026, http://unityqueen.com/2026/08/24/real-time-volumetric-fog-in-unity-using-3d-textures-and-compute-shaders/, fetched 2026-10-29 (unofficial)
[S5] PDF Real-Time Light Transport in Analytically Integrable Participating Media, https://cgg.mff.cuni.cz/~tomas/publications/pub-thesis-real-time-scattering/thesis.pdf, fetched 2026-10-29 (unofficial)
