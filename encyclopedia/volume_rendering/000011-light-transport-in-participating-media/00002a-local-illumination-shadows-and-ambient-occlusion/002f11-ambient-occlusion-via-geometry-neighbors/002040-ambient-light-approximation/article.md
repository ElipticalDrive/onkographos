# Ambient Light Approximation

Also called: Ambient Lighting for Volume Clouds, Sky Ambient, Upwards Ray Approximation

## Overview
Ambient Light Approximation provides a computationally efficient estimate of indirect sky illumination in participating media, primarily volumetric clouds, by replacing the integration of the sky hemisphere with a single upward ray [S1, unofficial]. Unlike full path tracing or multi-scattering models, it simulates the shadowing effect of cloud density on ambient light without the noise or cost of random direction sampling [S1, unofficial]. It sets itself apart from siblings like [Lighting Models](2056-lighting-models.md) by specifically addressing the "dark edge" and blue tint artifacts caused by ignoring indirect light, using a pre-integrated sky value modulated by local density [S2, unofficial].

## How it works
The method approximates the integral of sky radiance over the upper hemisphere by evaluating a single ray directed upwards from the shading point. This approach relies on the assumption that the sky is the dominant source of ambient light and that the "blurry" nature of volumetric scattering allows for significant simplification [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   $\mathbf{p}$: Current sample position in the volume.
    *   $\sigma_t(\mathbf{x})$: Extinction coefficient field (density).
    *   $L_{sky}$: Pre-integrated ambient sky color/value.
    *   $D_{up}(\mathbf{p})$: Pre-calculated or runtime-computed optical depth along the upward ray from $\mathbf{p}$.
    *   $profile(\mathbf{p})$: A scalar representing the sample's position relative to the cloud boundary (e.g., distance to edge or normalized depth) [S1, unofficial].
*   **Outputs:**
    *   $L_{ambient}$: The estimated ambient radiance contribution at $\mathbf{p}$.

### Data Structures
*   **Sky Light Value ($L_{sky}$):** A single RGB color vector or scalar intensity. It can be a fixed artist-defined color, a result of spherical harmonics integration, or a sample from a pre-computed SkyLight LUT (e.g., from atmospheric scattering passes) [S1, unofficial][S2, unofficial].
*   **Ambient Density Field ($D_{up}$):** A lower-resolution 3D texture or grid storing the integrated density along the upward vector for each voxel. This field is static for static clouds and can be pre-calculated because the data is naturally smooth [S1, unofficial]. If not pre-calculated, it is computed via a short ray march upwards [S1, unofficial].
*   **Cloud LUT (Optional):** In some implementations (e.g., UE4-based), an Ambient Occlusion factor is stored in a Cloud LUT (specifically the B channel), modulated by altitude [S2, unofficial].

### Steps
1.  **Pre-calculate Sky Light:** Integrate the sky hemisphere into a single value $L_{sky}$ using random sampling, spherical harmonics, or a fixed color [S1, unofficial]. If using a dynamic atmosphere, sample a pre-baked SkyLight LUT or Ambient Probe (e.g., 3rd order SH encoding) [S2, unofficial].
2.  **Evaluate Upward Transmittance:** Determine the transmittance from the sky to the current point $\mathbf{p}$ by sampling the pre-calculated $D_{up}(\mathbf{p})$ field. The transmittance is $T_{up} = \exp(-D_{up}(\mathbf{p}))$ [S1, unofficial].
3.  **Compute Shadowing Factor:** Calculate a shadowing factor based on the sample's position within the cloud volume. This is often modeled using a "profile" function (e.g., square root of normalized depth) to ensure edges receive more ambient light than the center [S1, unofficial]. Let this be $S_{profile} = \text{saturate}(profile(\mathbf{p})^\gamma)$, where $\gamma$ is typically 0.5 [S1, unofficial].
4.  **Calculate Ambient Contribution:** Combine the sky light, transmittance, and shadowing factor:
    $$ L_{ambient} = L_{sky} \cdot T_{up} \cdot S_{profile} $$
    In implementations using Cloud LUTs, an additional AO factor $AO_{lut}$ is applied based on altitude and the LUT's B channel:
    $$ L_{ambient} = L_{sky} \cdot T_{up} \cdot AO_{lut} $$
    [S2, unofficial]
5.  **Correct Color Tint:** To counteract the blue tint caused by ignoring multiple scattering (which makes clouds appear too blue), apply a saturation adjustment to $L_{sky}$ or the final $L_{ambient}$ [S2, unofficial].
6.  **Accumulate:** Add $L_{ambient}$ to the direct light contribution, scaled by the sample's density $\sigma_t(\mathbf{p})$, step size $\Delta s$, and the ray's transmittance from the camera $T_{cam}$:
    $$ L_{out} += L_{ambient} \cdot \sigma_t(\mathbf{p}) \cdot \Delta s \cdot T_{cam} $$
    [S1, unofficial]

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_ambient_light(p, sigma_t, step_size, transmittance_from_camera, 
                          sky_light_color, ambient_density_field, profile_value, 
                          use_cloud_lut=False, cloud_lut_ao=1.0, saturation_factor=1.0):
    """
    Computes ambient light approximation for a volume sample.
    
    Args:
        p: Current 3D position.
        sigma_t: Extinction coefficient at p.
        step_size: Length of the current ray march step.
        transmittance_from_camera: Transmittance from camera to p.
        sky_light_color: Pre-integrated sky ambient color (RGB).
        ambient_density_field: 3D texture/grid storing upward integrated density.
        profile_value: Normalized distance to cloud edge or depth profile (0-1).
        use_cloud_lut: Boolean flag for using Cloud LUT AO.
        cloud_lut_ao: Ambient occlusion factor from Cloud LUT B channel.
        saturation_factor: Artist parameter to correct blue tint.
        
    Returns:
        Ambient radiance contribution to be added to the sample.
    """
    # Step 1: Get upward transmittance from pre-calculated density field
    # D_up is the integrated density along the upward ray from p
    d_up = ambient_density_field.sample(p)
    transmittance_up = exp(-d_up)
    
    # Step 2: Calculate shadowing based on profile (edge vs center)
    # Using power 0.5 as per common approximation for falloff
    profile_shadow = saturate(pow(profile_value, 0.5))
    
    # Step 3: Determine final AO factor
    if use_cloud_lut:
        # AO varies with altitude and is modulated by Cloud LUT B channel
        # Assuming cloud_lut_ao already incorporates altitude gradient logic
        ao_factor = cloud_lut_ao
    else:
        ao_factor = profile_shadow
        
    # Step 4: Combine sky light with transmittance and AO
    ambient_light = sky_light_color * transmittance_up * ao_factor
    
    # Step 5: Correct color saturation to mitigate blue tint from lack of multiple scattering
    ambient_light = adjust_saturation(ambient_light, saturation_factor)
    
    # Step 6: Scale by local density, step size, and camera transmittance
    contribution = ambient_light * sigma_t * step_size * transmittance_from_camera
    
    return contribution
```

### Parameters and Typical Values
*   **Profile Power:** 0.5 is commonly used to soften the falloff of ambient light at cloud edges [S1, unofficial].
*   **Saturation Factor:** An artist-tunable parameter, typically $< 1.0$ to reduce the blue tint [S2, unofficial].
*   **AO at Cloud Bottom:** User-defined value, often low (e.g., 0.2-0.5), increasing to 1.0 at higher altitudes [S2, unofficial].

### Edge Cases and Pitfalls
*   **Blue Tint:** Ignoring multiple scattering causes clouds to appear unnaturally blue. This must be corrected via saturation adjustment or by blending in a secondary light source [S2, unofficial].
*   **Resolution of $D_{up}$:** The upward density field can be stored at a lower resolution than the main volume because the data is naturally blurry. However, if too low, it may cause banding or loss of detail in shadowing [S1, unofficial].
*   **Static vs. Dynamic:** The $D_{up}$ field is pre-calculated for static grids. For dynamic clouds, it must be recomputed or approximated, which increases cost [S1, unofficial].

## Subtopics

### Bruneton-style Sky Irradiance
An atmosphere pass that builds a sky irradiance table to provide sky light from each surface's own sky, ensuring ground seen from orbit keeps its sky fill and terrain past the terminator is not lit by the eye's daylight [S3, unofficial]. It covers 16 altitudes by 24 sun cosines, with 6 symmetric L2 harmonics per entry in the local frame, published as a uniform `atmosphere_sky_table` [S3, unofficial]. Deferred lighting looks up each pixel's own altitude and sun angle from this table [S3, unofficial]. Performance cost is approximately 0.60 ms for the deferred lighting pass [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample if $D_{up}$ is pre-calculated and stored in a texture. If computed via ray march, it adds a small constant number of steps (typically short upward ray).
*   **Memory Complexity:** $O(N)$ for the $D_{up}$ field, where $N$ is the number of voxels. Can be reduced by using a lower-resolution grid [S1, unofficial].
*   **Performance:** Very fast compared to full multi-scattering or path tracing. The Bruneton-style variant adds ~0.60 ms for deferred lighting lookups [S3, unofficial].
*   **Strengths:** Efficient, visually plausible for large-scale clouds, avoids noise associated with random sampling [S1, unofficial].
*   **Limitations:** Cannot accurately represent complex indirect lighting interactions (e.g., light bouncing between clouds and ground), leading to blue tint and lack of multiple scattering effects [S2, unofficial]. Not suitable for scenes requiring high-fidelity global illumination.

## Background
The method originates from practical real-time rendering techniques for volumetric clouds, where full global illumination is too expensive. It was popularized in game development contexts, such as Unreal Engine 4 atmospheric scattering implementations and custom cloud renderers [S1, unofficial][S2, unofficial]. The Bruneton-style irradiance table approach is derived from precomputed radiance transfer techniques for planetary atmospheres [S3, unofficial].

## Sources
[S1] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
[S2] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S3] Voxel planet: chunk trees, a fully destructible planet, and the lighting to show it, https://github.com/Far-Beyond-Pulsar/Helio/pull/327, fetched 2026-10-28 (unofficial)
