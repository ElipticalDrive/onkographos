# Ambient Light Approximation

Also called: Ambient Lighting for Volume Clouds, Sky Ambient, Upwards Ray Approximation

## Overview
Ambient Light Approximation simulates indirect sky illumination in participating media by integrating the entire sky hemisphere into a single upward-pointing light value, weighted by local cloud density to simulate self-shadowing [S1, unofficial]. Unlike full multiple scattering or complex irradiance tables, it uses a single precalculated or sampled upward ray to approximate the diffuse fill light, preventing the underside of clouds from appearing unnaturally dark [S1, unofficial]. It differs from [Ambient Occlusion via Geometry/Neighbors](5936-ambient-occlusion-via-geometry-neighbors.md) by relying on volumetric density rather than surface geometry, and from [Bruneton-style Sky Irradiance](#bruneton-style-sky-irradiance) by using a simplified single-ray model instead of a multi-dimensional lookup table [S3, unofficial].

## How it works
**Inputs:**
*   Current sample position in the volume.
*   Cloud density profile or distance-to-edge field (to determine local opacity/shadowing).
*   Ambient light color/intensity (either a fixed color, pre-integrated sky value, or sampled from a LUT/probe) [S1, S2, unofficial].
*   Transmissance accumulated so far along the view ray.
*   Step size of the current ray march.

**Outputs:**
*   Added ambient light contribution to the current sample's light accumulation.

**Data Structures:**
*   **Sky Light Value:** A single RGB color vector representing the integrated sky hemisphere. This can be a constant, a value from a pre-integrated sky model, or a sample from a `SkyLightLut` [S1, S2, unofficial].
*   **Ambient Density Field:** A lower-resolution grid or texture storing the summed density along the upward ray for each voxel. This is precalculated for static grids [S1, unofficial].
*   **Ambient Occlusion (AO) Map:** A texture (e.g., Cloud Lut B channel) storing altitude-based AO values [S2, unofficial].

**Steps:**
1.  **Determine Ambient Light Color:**
    *   If atmospheric scattering is enabled, sample the `SkyLightLut` (generated via UE4 atmospheric scattering methods) to get the ambient light color [S2, unofficial].
    *   If atmospheric scattering is disabled, sample a baked Ambient Probe encoded with 3rd-order Spherical Harmonics (SH) [S2, unofficial].
    *   Alternatively, use a fixed color resembling the sky [S1, unofficial].

2.  **Calculate Shadowing Factor:**
    *   Compute the shadowing factor based on the accumulated density along an upward ray from the current sample position.
    *   Let $D_{up}$ be the summed density of the upward ray. This value can be retrieved from a precalculated texture or computed by marching a single ray upward [S1, unofficial].
    *   The shadowing factor is typically modeled as $\exp(-D_{up})$ [S1, unofficial].
    *   Optionally, modulate this by a profile function (e.g., $\sqrt{\text{profile}}$) to adjust falloff at cloud edges [S1, unofficial].

3.  **Calculate Altitude-Based AO (Optional/Engine Specific):**
    *   Determine an altitude factor that starts at a user-defined value at the cloud bottom and increases to 1 as altitude increases [S2, unofficial].
    *   Multiply this factor by the Cloud Lut B channel's AO value [S2, unofficial].

4.  **Compute Ambient Contribution:**
    *   Combine the ambient light color, shadowing factor, and AO.
    *   Formula:
        $$ L_{ambient} = L_{sky} \cdot \exp(-D_{up}) \cdot AO_{factor} $$
    *   Where $L_{sky}$ is the ambient light color, $D_{up}$ is the upward density, and $AO_{factor}$ is the altitude-based AO [S1, S2, unofficial].

5.  **Accumulate:**
    *   Add the ambient contribution to the sample light, scaled by transmissance, sample density, and step size.
    *   Formula:
        $$ L_{total} += L_{ambient} \cdot T \cdot \rho \cdot \Delta s $$
    *   Where $T$ is transmissance, $\rho$ is sample density, and $\Delta s$ is step size [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ambient_light(sample_pos, transmissance, sample_density, step_size, ambient_light_color):
    # 1. Get upward density (precalculated or sampled)
    # In a real implementation, this might be a texture lookup or a single ray march
    upward_density = get_upward_density(sample_pos)
    
    # 2. Calculate shadowing factor
    # Using exponential attenuation based on upward density
    shadow_factor = exp(-upward_density)
    
    # Optional: Modulate by profile for edge falloff
    # profile_value = get_cloud_profile(sample_pos)
    # shadow_factor *= sqrt(profile_value)
    
    # 3. Get AO factor (if using altitude-based AO)
    # altitude_factor = map_altitude_to_ao(sample_pos.y) # 0 at bottom, 1 at top
    # ao_texture_value = sample_cloud_lut_b(sample_pos)
    # ao_factor = altitude_factor * ao_texture_value
    
    # For simplicity, assuming ao_factor is 1 if not using altitude AO
    ao_factor = 1.0 
    
    # 4. Compute ambient light contribution
    ambient_contribution = ambient_light_color * shadow_factor * ao_factor
    
    # 5. Accumulate into total light
    # Note: In a ray marching loop, this is added to the current sample's light
    # before being multiplied by transmissance and density for the final color
    return ambient_contribution * transmissance * sample_density * step_size
```

**Parameters:**
*   **Ambient Light Color:** Can be a fixed RGB value or sampled from a LUT [S1, S2, unofficial].
*   **Saturation Adjustment:** An artist parameter to adjust the saturation of the ambient light color to correct for an overly blue tint caused by ignoring multiple scattering [S2, unofficial].
*   **Cloud Bottom AO Value:** A user-defined value for the AO factor at the cloud bottom [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Blue Tint:** Ignoring multiple scattering in ambient light can cause clouds to appear too blue [S2, unofficial]. Mitigation: Adjust ambient light saturation [S2, unofficial].
*   **Performance:** Shooting actual rays in random directions for ambient light is too expensive or noisy [S1, unofficial]. The upward ray approximation is necessary for performance.
*   **Resolution:** The upward density data is naturally blurry and can be stored in a lower-resolution grid without significant visual loss [S1, unofficial].

## Subtopics

### Bruneton-style Sky Irradiance
An atmosphere pass that builds a sky irradiance table to provide sky light from each surface's own sky, ensuring ground seen from orbit keeps its sky fill and terrain past the terminator is not lit by the eye's daylight [S3, unofficial]. It differs from the single-ray approximation by using a multi-dimensional lookup table [S3, unofficial].
*   **Definition:** A deferred lighting technique that looks up each pixel's own altitude and sun angle from a precomputed sky irradiance table [S3, unofficial].
*   **Data Structure:** Sky irradiance table covers 16 altitudes by 24 sun cosines, with 6 symmetric L2 harmonics per entry in the local frame. Published as a uniform `atmosphere_sky_table` [S3, unofficial].
*   **Performance:** DeferredLight costs 0.60 ms [S3, unofficial].
*   **When to choose:** When high-fidelity sky lighting is required for terrain or surfaces at various altitudes and sun angles, especially in planetary rendering contexts [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample for the ambient contribution if the upward density is precalculated and stored in a texture. If the upward ray is marched per sample, it adds $O(N_{up})$ where $N_{up}$ is the number of steps in the upward ray [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for storing the precalculated upward density grid, where $V$ is the number of voxels. The grid can be lower resolution than the main volume due to the blurry nature of the data [S1, unofficial].
*   **Measured Performance:** DeferredLight using Bruneton-style irradiance costs 0.60 ms [S3, unofficial].
*   **Strengths:** Computationally efficient compared to full multiple scattering or random ray sampling for ambient light [S1, unofficial]. Provides realistic shadowing for cloud undersides [S1, unofficial].
*   **Limitations:** Approximates the sky hemisphere as a single upward ray, which may not capture complex indirect lighting scenarios [S1, unofficial]. Can result in an overly blue tint if multiple scattering is not accounted for [S2, unofficial].
*   **When to use:** Real-time volume cloud rendering where performance is critical and full multiple scattering is too expensive [S1, S2, unofficial].
*   **When not:** When high-fidelity global illumination or complex indirect lighting effects are required [S3, unofficial].

## Background
The ambient light approximation for volume clouds was described by Jack Tollenaar in an unofficial article on volumetric cloud rendering [S1, unofficial]. It is also referenced in a Chinese article on volume cloud implementation, which discusses sampling from a SkyLightLut or Ambient Probe and adjusting for blue tint [S2, unofficial]. The Bruneton-style sky irradiance table is part of the Helio project's voxel planet lighting, described in a GitHub pull request [S3, unofficial].

## Sources
[S1] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S2] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S3] Voxel planet: chunk trees, a fully destructible planet, and the lighting to show it, https://github.com/Far-Beyond-Pulsar/Helio/pull/327, fetched 2026-10-28 (unofficial)
