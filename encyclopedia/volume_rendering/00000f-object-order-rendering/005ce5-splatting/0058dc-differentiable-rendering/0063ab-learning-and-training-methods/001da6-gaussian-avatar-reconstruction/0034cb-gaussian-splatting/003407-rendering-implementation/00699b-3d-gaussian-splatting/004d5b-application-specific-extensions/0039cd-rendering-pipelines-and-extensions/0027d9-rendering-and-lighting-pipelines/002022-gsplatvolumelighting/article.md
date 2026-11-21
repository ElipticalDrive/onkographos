# GsplatVolumeLighting

Also called: Gaussian Splat Volume Ray Marching, Volume Occlusion for 3DGS

## Overview
GsplatVolumeLighting is a volume lighting and occlusion technique for 3D Gaussian Splatting (3DGS) that ray-marches through a 3D density texture to compute transmittance for shadow and volume occlusion, combined with per-light attenuation and diffuse shading [S1, unofficial]. It is implemented within the per-splat lighting evaluation loop using a shared 3D density texture [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `_GsplatVolumeDensity`: A `sampler3D` texture storing density. The R8 channel encodes $\sqrt{\text{normalized optical depth}}$ [S1, unofficial].
    *   `_GsplatVolumeOcclusionParams`: A `float4` packing:
        *   `x`: Optical-depth scale (0 disables occlusion) [S1, unofficial].
        *   `y`: Number of sample steps [S1, unofficial].
        *   `z`: Start distance [S1, unofficial].
    *   `_GsplatVolumeBoundsMin` / `_GsplatVolumeBoundsMax`: `float4` values defining the axis-aligned volume bounds in local space [S1, unofficial].
    *   `_GsplatVolumeBoundsInvSize`: `float4` containing the inverse dimensions of the volume bounds [S1, unofficial].
    *   `_GsplatVolumeEnabled`: Boolean flag (0 disables lighting) [S1, unofficial].
    *   `_GsplatVolumeAmbientColor`: `float4` ambient color [S1, unofficial].
    *   `_GsplatVolumeLightingParams`: `float4` packing:
        *   `y`: Ambient intensity multiplier [S1, unofficial].
        *   `z`: Direct lighting intensity multiplier [S1, unofficial].
    *   `_GsplatVolumeLightColors`: Array of light colors [S1, unofficial].
    *   Light properties: `lightType` (directional < 0.5, point/spot > 0.5), `lightPosition` (w > 0.5 for point lights), `lightDirection`, `lightDirection.w` (range fade parameter), `spotParams` (x: min cosine, y: exponent) [S1, unofficial].
    *   Splat data: local center, local normal, and world view direction [S1, unofficial].
*   **Outputs:**
    *   Final lighting value (float), typically added to the splat's base color or used as a multiplier [S1, unofficial].

**Data Structures:**
*   **Volume Texture:** `sampler3D` with R8 format. Values are squared during sampling to recover optical depth contribution [S1, unofficial].
*   **Constants:** `GSPLAT_VOLUME_OPAQUE_DEPTH = 8.0`. Transmittance below $\exp(-8)$ is treated as fully opaque [S1, unofficial].

**Steps:**

1.  **Ray-Box Intersection:**
    *   Compute `directionSign` using `lerp(-1.0, 1.0, step(0.0, direction))` [S1, unofficial].
    *   Clamp the ray direction magnitude to $10^{-5}$ to prevent division by zero [S1, unofficial].
    *   Compute entry (`t0`) and exit (`t1`) distances against `boundsMin` and `boundsMax` using `directionSign` [S1, unofficial].
    *   Return `max(0.0, min(farDistance.x, min(farDistance.y, farDistance.z)))` as the maximum ray length `rayLength` [S1, unofficial].

2.  **Light Direction Transformation:**
    *   Transform the light direction to local space using `_GsplatVolumeWorldToLocal` [S1, unofficial].
    *   Normalize the result. If length $\le 10^{-6}$, return transmittance 1.0 [S1, unofficial].

3.  **Ray Length Clamping (Point Lights):**
    *   If `lightPosition.w > 0.5` (point light), clamp `rayLength` to the Euclidean distance from `localCenter` to `lightPosition` [S1, unofficial].

4.  **Usable Length Check:**
    *   Calculate `usableLength = rayLength - startDistance` (where `startDistance` is `_GsplatVolumeOcclusionParams.z`) [S1, unofficial].
    *   If `usableLength <= 10^{-5}`, return transmittance 1.0 [S1, unofficial].

5.  **Sample Count and Step Size:**
    *   Determine `sampleCount = clamp((int)round(_GsplatVolumeOcclusionParams.y), 1, 24)` [S1, unofficial].
    *   Compute `stepLength = usableLength / sampleCount` [S1, unofficial].

6.  **Initialization:**
    *   Initialize UVW coordinates at the midpoint of the first step:
        $$uvw = \frac{\text{localCenter} + \text{localDirection} \cdot (\text{startDistance} + 0.5 \cdot \text{stepLength}) - \text{boundsMin}}{\text{boundsSize}}$$
        where `boundsSize` is derived from `_GsplatVolumeBoundsInvSize` (inverse dimensions) [S1, unofficial].
    *   Compute step delta: `uvwStep = localDirection * stepLength * invSize` [S1, unofficial].
    *   Initialize `opticalDepth = 0.0` [S1, unofficial].

7.  **Ray Marching Loop:**
    *   Loop `sampleIndex` from 0 to `sampleCount - 1` (max 24) [S1, unofficial].
    *   **Early Exit:** If `opticalDepth >= GSPLAT_VOLUME_OPAQUE_DEPTH / strength` (where `strength` is `_GsplatVolumeOcclusionParams.x`), break loop [S1, unofficial].
    *   **Boundary Check:** If `uvw` is outside $[0, 1]$ in any dimension, skip sampling for this step [S1, unofficial].
    *   **Sampling:** Sample the R8 channel of `_GsplatVolumeDensity`. Square the value ($d_i^2$) and add to `opticalDepth` [S1, unofficial].
    *   **Advance:** `uvw += uvwStep` [S1, unofficial].

8.  **Transmittance Calculation:**
    *   Compute transmittance $T$:
        $$T = \exp(-\text{strength} \cdot \text{opticalDepth})$$
        where `strength` is `_GsplatVolumeOcclusionParams.x` and `opticalDepth` is the accumulated sum of squared samples [S1, unofficial].

9.  **Per-Splat Lighting Evaluation:**
    *   If `_GsplatVolumeEnabled == 0`, return 1.0 [S1, unofficial].
    *   Transform splat `center` to world space [S1, unofficial].
    *   Transform and normalize splat `normal`. If $\text{dot}(\text{worldNormal}, \text{viewDirection}) < 0.0$, flip the normal [S1, unofficial].
    *   Compute `ambient = _GsplatVolumeAmbientColor.rgb * _GsplatVolumeLightingParams.y` [S1, unofficial].
    *   Initialize `direct = 0.0` [S1, unofficial].

10. **Per-Light Processing (Max 4 Lights):**
    *   Compute vector `toLight` and distance [S1, unofficial].
    *   **Attenuation:**
        *   If `lightType < 0.5` (directional), `attenuation = 1.0` [S1, unofficial].
        *   If `lightType > 0.5` (point/spot):
            *   Compute `distanceSquared` [S1, unofficial].
            *   Compute `normalizedDistanceSquared = distanceSquared * lightDirection.w` [S1, unofficial].
            *   Compute `rangeFade = saturate(1.0 - \text{normalizedDistanceSquared}^2)` [S1, unofficial].
            *   `attenuation = \text{rangeFade}^2 / distanceSquared` [S1, unofficial].
    *   **Spot Cone Falloff (if `lightType > 1.5`):**
        *   Compute `spotCosine = dot(lightDirection.xyz, -toLight)` [S1, unofficial].
        *   Compute `spot = saturate((\text{spotCosine} - \text{spotParams.x}) \cdot \text{spotParams.y})` [S1, unofficial].
        *   `attenuation *= \text{spot}^2` [S1, unofficial].
    *   **Visibility:**
        *   Compute `visibility` using the transmittance $T$ calculated in Step 8 [S1, unofficial].
        *   **Pitfall Mitigation:** If irradiance (light intensity before visibility) is $\le 0.0$, skip transmittance calculation to avoid expensive shadow marches for back-facing or out-of-range lights [S1, unofficial].
        *   Add to `direct`: `direct += _GsplatVolumeLightColors[lightIndex].rgb * (irradiance * visibility)` [S1, unofficial].

11. **Final Output:**
    *   Scale `direct` by `_GsplatVolumeLightingParams.z` [S1, unofficial].
    *   Return `max(ambient + direct, 0.0)` [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gsplat_volume_lighting(splat_center_local, splat_normal_local, view_direction, 
                           light_direction_world, light_position, light_type, 
                           light_color, light_direction_w, spot_params,
                           volume_density_sampler, volume_occlusion_params, 
                           volume_bounds_min, volume_bounds_max, volume_bounds_inv_size,
                           volume_ambient_color, volume_lighting_params, volume_enabled):
    """
    Computes volume lighting and occlusion for a single 3D Gaussian Splat.
    """
    if not volume_enabled:
        return 1.0

    # 1. Ray-Box Intersection
    direction = normalize(light_direction_world)
    dir_len = length(direction)
    if dir_len < 1e-5:
        return 1.0
    direction = direction / max(dir_len, 1e-5)
    
    # directionSign = lerp(-1.0, 1.0, step(0.0, direction))
    direction_sign = where(direction >= 0.0, 1.0, -1.0)
    
    # Compute t0/t1 against boundsMin/BoundsMax
    t0 = (volume_bounds_min - splat_center_local) / direction
    t1 = (volume_bounds_max - splat_center_local) / direction
    t_near = max(min(t0.x, t1.x), min(t0.y, t1.y), min(t0.z, t1.z))
    t_far = min(max(t0.x, t1.x), max(t0.y, t1.y), max(t0.z, t1.z))
    
    # Return max(0.0, min(farDistance.x, min(farDistance.y, farDistance.z)))
    ray_length = max(0.0, min(t_far.x, min(t_far.y, t_far.z)))

    # 2. Light Direction Transformation
    # Transform light direction to local space using _GsplatVolumeWorldToLocal
    # (Not given in the sources. Standard inverse rotation matrix multiplication.)
    local_light_direction = light_direction_world 
    local_light_direction = normalize(local_light_direction)
    if length(local_light_direction) <= 1e-6:
        return 1.0

    # 3. Ray Length Clamping (Point Lights)
    if light_position.w > 0.5:
        dist_to_light = length(light_position - splat_center_local)
        ray_length = min(ray_length, dist_to_light)

    # 4. Usable Length Check
    start_distance = volume_occlusion_params.z
    usable_length = ray_length - start_distance
    if usable_length <= 1e-5:
        return 1.0

    # 5. Sample Count and Step Size
    strength = volume_occlusion_params.x
    if strength == 0:
        return 1.0 # Occlusion disabled
    
    sample_count = clamp(round(volume_occlusion_params.y), 1, 24)
    step_length = usable_length / sample_count

    # 6. Initialization
    # UVW at midpoint of first step
    uvw = (splat_center_local + local_light_direction * (start_distance + 0.5 * step_length) - volume_bounds_min) * volume_bounds_inv_size
    uvw_step = local_light_direction * step_length * volume_bounds_inv_size
    
    optical_depth = 0.0
    opaque_threshold = 8.0 / strength

    # 7. Ray Marching Loop
    for i in range(sample_count):
        if optical_depth >= opaque_threshold:
            break
        
        # Boundary Check
        if uvw.x < 0.0 or uvw.x > 1.0 or uvw.y < 0.0 or uvw.y > 1.0 or uvw.z < 0.0 or uvw.z > 1.0:
            uvw += uvw_step
            continue
            
        # Sampling
        d = texture3D(volume_density_sampler, uvw).r
        optical_depth += d * d
        
        uvw += uvw_step

    # 8. Transmittance Calculation
    transmittance = exp(-strength * optical_depth)

    # 9. Per-Splat Lighting Evaluation
    # Transform center and normal to world space
    # (Not given in the sources. Standard transformation matrix multiplication.)
    center_world = splat_center_local 
    normal_world = normalize(splat_normal_local)
    
    # Flip normal if back-facing
    if dot(normal_world, view_direction) < 0.0:
        normal_world = -normal_world

    ambient = volume_ambient_color.rgb * volume_lighting_params.y
    direct = 0.0

    # 10. Per-Light Processing
    # Compute toLight and distance
    to_light = light_position - center_world
    distance_squared = dot(to_light, to_light)
    
    # Irradiance calculation (simplified for single light)
    irradiance = 1.0 # Placeholder for light intensity/color magnitude
    
    # Pitfall Mitigation: Skip if irradiance <= 0.0
    if irradiance <= 0.0:
        visibility = 1.0
    else:
        visibility = transmittance

    # Attenuation
    if light_type < 0.5:
        attenuation = 1.0
    else:
        normalized_distance_squared = distance_squared * light_direction_w
        range_fade = saturate(1.0 - normalized_distance_squared * normalized_distance_squared)
        attenuation = (range_fade * range_fade) / distance_squared

    # Spot Cone Falloff
    if light_type > 1.5:
        spot_cosine = dot(normalize(light_direction_world), -normalize(to_light))
        spot = saturate((spot_cosine - spot_params.x) * spot_params.y)
        attenuation *= spot * spot

    direct += light_color.rgb * (irradiance * visibility * attenuation)

    # 11. Final Output
    direct *= volume_lighting_params.z
    return max(ambient + direct, 0.0)
```

## Operating characteristics
*   **Time Complexity:** $O(N)$ per splat per light, where $N$ is the number of samples (max 24). The early exit on `opticalDepth >= GSPLAT_VOLUME_OPAQUE_DEPTH / strength` can significantly reduce average case performance [S1, unofficial].
*   **Memory Complexity:** $O(V)$ for the 3D density texture, where $V$ is the volume resolution. The texture is shared across splats [S1, unofficial].
*   **Performance:** The method is designed for real-time rendering in Unity 6, leveraging GPU compute and texture sampling [S1, unofficial].
*   **Strengths:** Provides volumetric occlusion and lighting for 3DGS, enhancing realism. Supports Built-in Render Pipeline, URP, and HDRP [S1, unofficial].
*   **Limitations:** Limited to 4 lights. Max 24 samples per ray march. Requires a 3D density texture [S1, unofficial].
*   **When to Use:** When volumetric occlusion and lighting are required for 3DGS content in Unity 6 [S1, unofficial].
*   **When Not to Use:** When performance is critical and volumetric effects are not needed, or when more than 4 lights are required [S1, unofficial].

## Background
GsplatVolumeLighting is part of the UnitySplats package, a cross-platform Unity 6 package for importing, loading, and rendering 3D Gaussian Splatting content [S1, unofficial]. It is a modified and substantially extended fork of `wuyize25/gsplat-unity` [S1, unofficial]. The method is used for volume occlusion and lighting of 3DGS content in Unity 6 [S1, unofficial].

## Sources
[S1] arloopa/UnitySplats, https://github.com/arloopa/UnitySplats, fetched 2026-10-27 (unofficial)
