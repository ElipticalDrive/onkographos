# GsplatVolumeLighting

Also called: Gaussian Splat Volume Ray Marching, Volume Occlusion for 3DGS

## Overview
GsplatVolumeLighting is a volume lighting and occlusion technique for 3D Gaussian Splatting (3DGS) that ray-marches through a 3D density texture to compute transmittance for shadow and volume occlusion, combined with per-light attenuation and diffuse shading [S1, unofficial]. It extends the standard 3DGS rendering pipeline by introducing volumetric shadow casting, allowing splats to cast and receive shadows from a defined density volume rather than relying solely on geometric primitives or screen-space techniques. Unlike [Duplex-GS](1988-duplex-gs.md) which focuses on dual-visibility for large scenes, or [4DGS-Volume-Cloud Two-Stage Relighting](4101-4dgs-volume-cloud-two-stage-relighting.md) which uses a two-stage cloud approach, GsplatVolumeLighting integrates volume occlusion directly into the per-splat lighting evaluation using a unified ray-marching step [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Splat attributes: Position $\mathbf{p}_i$, Normal $\mathbf{n}_i$, View Direction $\mathbf{v}_i$.
    *   Volume Data: A 3D texture `_GsplatVolumeDensity` (R8 channel) storing density.
    *   Volume Bounds: Axis-aligned bounding box defined by `_GsplatVolumeBoundsMin` and `_GsplatVolumeBoundsMax` in local space.
    *   Light Data: Up to 4 lights, each with type (directional, point, spot), position/direction, color, and attenuation parameters.
    *   Parameters: `_GsplatVolumeOcclusionParams` (optical-depth scale, sample steps, start distance), `_GsplatVolumeAmbientColor`, `_GsplatVolumeLightingParams`.
*   **Output:**
    *   Final lighting value (scalar or RGB) for the splat, combining ambient and direct volumetric lighting.

**Data Structures:**
*   `_GsplatVolumeDensity`: `sampler3D` texture. The R8 channel encodes $\sqrt{\text{normalized optical depth}}$ [S1, unofficial].
*   `_GsplatVolumeOcclusionParams`: `float4` packing:
    *   $x$: Optical-depth scale (0 disables volume).
    *   $y$: Sample steps (clamped 1–24).
    *   $z$: Start distance.
    *   $w$: Unused/Reserved.
*   `_GsplatVolumeBoundsMin/Max`: `float4` defining the AABB in local space.
*   `_GsplatVolumeBoundsInvSize`: `float4` containing the inverse dimensions of the volume bounds ($1 / (\text{Max} - \text{Min})$).
*   `GSPLAT_VOLUME_OPAQUE_DEPTH`: Constant $8.0$. Transmittance below $\exp(-8)$ is treated as fully opaque [S1, unofficial].

**Algorithm Steps:**

1.  **Volume Ray-Box Intersection:**
    Calculate the entry ($t_0$) and exit ($t_1$) distances of the ray from the volume bounds.
    *   Compute `directionSign`: $\text{lerp}(-1.0, 1.0, \text{step}(0.0, \text{direction}))$.
    *   Clamp ray direction magnitude to $10^{-5}$ to prevent division by zero [S1, unofficial].
    *   Compute $t_0, t_1$ against `boundsMin` and `boundsMax`.
    *   Return $\max(0.0, \min(\text{farDistance}.x, \min(\text{farDistance}.y, \text{farDistance}.z)))$ as the max valid ray length [S1, unofficial].

2.  **Light Direction Transformation:**
    Transform the light direction to the local space of the volume using `_GsplatVolumeWorldToLocal`. Normalize it. If the length is $\le 10^{-6}$, return transmittance $1.0$ [S1, unofficial].

3.  **Ray Length Clamping:**
    *   If the light is a point light (`lightPosition.w > 0.5`), clamp the ray length to the Euclidean distance from the local center to the light position.
    *   Calculate `usableLength` = `rayLength` - `startDistance`.
    *   If `usableLength` $\le 10^{-5}$, return transmittance $1.0$ [S1, unofficial].

4.  **Sampling Configuration:**
    *   Determine `sampleCount` = $\text{clamp}(\text{round}(\_GsplatVolumeOcclusionParams.y), 1, 24)$.
    *   Compute `stepLength` = `usableLength` / `sampleCount` [S1, unofficial].

5.  **Initialization:**
    *   Initialize UVW coordinates at the midpoint of the first step:
        $$ \text{uvw} = (\text{localCenter} + \text{localDirection} \cdot (\text{startDistance} + 0.5 \cdot \text{stepLength}) - \text{boundsMin}) \cdot \text{invSize} $$
    *   Compute step delta: $\text{uvwStep} = \text{localDirection} \cdot \text{stepLength} \cdot \text{invSize}$ [S1, unofficial].

6.  **Ray Marching Loop:**
    *   Initialize `opticalDepth` = $0.0$.
    *   Loop `sampleIndex` from $0$ to `sampleCount` - 1:
        *   If `sampleIndex` $\ge$ `sampleCount` or `opticalDepth` $\ge$ `GSPLAT_VOLUME_OPAQUE_DEPTH`, break.
        *   If `uvw` is outside $[0, 1]$ bounds, skip sample.
        *   Sample R8 channel $d_i$ from `_GsplatVolumeDensity`.
        *   Add $d_i^2$ to `opticalDepth`.
        *   Advance `uvw` += `uvwStep` [S1, unofficial].

7.  **Transmittance Calculation:**
    Compute transmittance $T$ using the formula:
    $$ T = \exp(-\text{strength} \cdot \text{opticalDepth}) $$
    where `strength` is `_GsplatVolumeOcclusionParams.x` [S1, unofficial].

8.  **Lighting Evaluation:**
    *   If `_GsplatVolumeEnabled` == 0, return $1.0$.
    *   Transform splat center to world space.
    *   Transform and normalize normal $\mathbf{n}$. Flip if $\mathbf{n} \cdot \mathbf{v}_i < 0.0$.
    *   Compute ambient: $\text{ambient} = \_GsplatVolumeAmbientColor.rgb \cdot \_GsplatVolumeLightingParams.y$.
    *   Initialize `direct` = $0.0$.
    *   For each light (max 4):
        *   Compute vector to light $\mathbf{l}$ and distance.
        *   Compute attenuation:
            *   Directional light (`lightType` < 0.5): attenuation = $1.0$.
            *   Point/Spot light:
                *   $\text{distanceSquared} = \|\mathbf{l}\|^2$.
                *   $\text{normalizedDistanceSquared} = \text{distanceSquared} \cdot \text{lightDirection}.w$.
                *   $\text{rangeFade} = \text{saturate}(1.0 - \text{normalizedDistanceSquared}^2)$.
                *   $\text{attenuation} = \text{rangeFade}^2 / \text{distanceSquared}$.
        *   Spot cone falloff (if `lightType` > 1.5):
            *   $\text{spotCosine} = \text{dot}(\text{lightDirection}.xyz, -\mathbf{l})$.
            *   $\text{spot} = \text{saturate}((\text{spotCosine} - \text{spotParams}.x) \cdot \text{spotParams}.y)$.
            *   $\text{attenuation} *= \text{spot}^2$.
        *   Compute visibility $V$ using the transmittance $T$ from step 7.
        *   Add to direct: $\text{direct} += \_GsplatVolumeLightColors[\text{lightIndex}].rgb \cdot (\text{irradiance} \cdot V)$.
        *   [G] Irradiance is assumed to be the light's intensity/color before attenuation.
    *   Scale `direct` by `_GsplatVolumeLightingParams.z`.
    *   Return $\max(\text{ambient} + \text{direct}, 0.0)$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gsplat_volume_lighting(splat, volume_texture, volume_bounds_min, volume_bounds_max, 
                           volume_inv_size, lights, params):
    """
    splat: dict with 'center', 'normal', 'view_direction'
    volume_texture: function sample_3d(uvw) returning float
    volume_bounds_min, volume_bounds_max: float3
    volume_inv_size: float3
    lights: list of light dicts
    params: dict with 'enabled', 'density_scale', 'sample_steps', 'start_dist', 
            'ambient_color', 'lighting_scale', 'opaque_depth'
    """
    if not params['enabled']:
        return 1.0

    # 1. Ray-Box Intersection
    ray_dir = splat['view_direction'] # Or light direction depending on context; 
                                      # here we assume ray from splat to light/volume
    # Clamp direction magnitude to prevent division by zero
    dir_len = length(ray_dir)
    if dir_len < 1e-5:
        return 1.0
    ray_dir = ray_dir / dir_len * max(dir_len, 1e-5)
    
    # Compute direction sign for slab intersection
    dir_sign = [ -1.0 if d < 0 else 1.0 for d in ray_dir ]
    
    # Compute t0 and t1 for each axis
    t0 = (volume_bounds_min - splat['center']) / ray_dir
    t1 = (volume_bounds_max - splat['center']) / ray_dir
    
    # Swap if necessary to ensure t0 < t1
    t_min = [min(t0[i], t1[i]) for i in range(3)]
    t_max = [max(t0[i], t1[i]) for i in range(3)]
    
    t_enter = max(t_min)
    t_exit = min(t_max)
    
    # Clamp to far distance (not specified, assume large value or t_exit)
    max_ray_len = t_exit 
    if max_ray_len < 0:
        return 1.0
        
    # 2. Light Direction Transformation (Local Space)
    # Assuming light direction is passed or computed in world space
    # Transform to local space using inverse world matrix (simplified as _GsplatVolumeWorldToLocal)
    local_light_dir = transform_to_local(splat['light_direction']) # Placeholder for actual transform
    local_light_dir = normalize(local_light_dir)
    if length(local_light_dir) <= 1e-6:
        return 1.0
        
    local_center = transform_to_local(splat['center'])
    
    # 3. Ray Length Clamping
    # If point light, clamp to distance to light
    if is_point_light(splat['light']):
        dist_to_light = length(local_center - splat['light_position'])
        max_ray_len = min(max_ray_len, dist_to_light)
        
    usable_length = max_ray_len - params['start_dist']
    if usable_length <= 1e-5:
        return 1.0
        
    # 4. Sampling Configuration
    sample_count = clamp(round(params['sample_steps']), 1, 24)
    step_length = usable_length / sample_count
    
    # 5. Initialization
    start_uv = local_center + local_light_dir * (params['start_dist'] + 0.5 * step_length)
    uvw = (start_uv - volume_bounds_min) * volume_inv_size
    uvw_step = local_light_dir * step_length * volume_inv_size
    
    # 6. Ray Marching Loop
    optical_depth = 0.0
    for i in range(sample_count):
        if optical_depth >= params['opaque_depth']:
            break
        if not (0.0 <= uvw.x <= 1.0 and 0.0 <= uvw.y <= 1.0 and 0.0 <= uvw.z <= 1.0):
            uvw += uvw_step
            continue
            
        # Sample R8 channel
        density_val = sample_3d(volume_texture, uvw)
        optical_depth += density_val ** 2
        uvw += uvw_step
        
    # 7. Transmittance Calculation
    transmittance = exp(-params['density_scale'] * optical_depth)
    
    # 8. Lighting Evaluation
    ambient = params['ambient_color'] * params['lighting_scale']
    direct = 0.0
    
    for light in lights:
        # Compute attenuation
        if light['type'] == 'directional':
            attenuation = 1.0
        else:
            dist_sq = length(light['to_light']) ** 2
            norm_dist_sq = dist_sq * light['direction'].w
            range_fade = clamp(1.0 - norm_dist_sq ** 2, 0.0, 1.0)
            attenuation = (range_fade ** 2) / dist_sq
            
        # Spot falloff
        if light['type'] == 'spot':
            spot_cosine = dot(light['direction'].xyz, -normalize(light['to_light']))
            spot = clamp((spot_cosine - light['spot_params'].x) * light['spot_params'].y, 0.0, 1.0)
            attenuation *= spot ** 2
            
        # Visibility
        visibility = transmittance
        
        # Add to direct
        direct += light['color'] * visibility * attenuation
        
    direct *= params['lighting_scale']
    return max(ambient + direct, 0.0)
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot S)$ per splat, where $N$ is the number of lights (max 4) and $S$ is the sample count (1–24). The ray-marching loop is fixed-size, making it predictable but potentially expensive for high-resolution volumes or many lights.
*   **Memory Complexity:** $O(V)$ for the 3D density texture, where $V$ is the volume resolution. The texture is typically R8, keeping memory footprint low.
*   **Performance:** Performance is heavily dependent on the volume resolution and the `sampleCount` parameter. Early exit on `opticalDepth >= GSPLAT_VOLUME_OPAQUE_DEPTH` mitigates over-marching in opaque regions [S1, unofficial].
*   **Strengths:** Provides physically plausible volumetric shadows for 3DGS, which is otherwise limited to surface-like rendering. Integrates seamlessly with Unity's lighting pipeline.
*   **Limitations:** Limited to 4 lights. Fixed sample count may lead to aliasing or noise if too low, or performance issues if too high. Requires a pre-computed 3D density texture.
*   **When to use:** When volumetric shadows are required for 3DGS content in Unity, especially for scenes with dense fog, smoke, or volumetric clouds.
*   **When not to use:** For real-time applications with strict performance budgets where volumetric shadows are not critical, or when the scene geometry is complex enough that geometric shadows are preferred.

## Background
GsplatVolumeLighting is part of the UnitySplats package, a cross-platform Unity 6 package for importing, loading, and rendering 3D Gaussian Splatting content [S1, unofficial]. It was developed by Arloopa as an extension to the original gsplat-unity renderer, which was a fork of wuyize25/gsplat-unity [S1, unofficial]. The technique addresses the limitation of standard 3DGS rendering, which does not natively support volumetric occlusion, by introducing a ray-marching step through a 3D density texture [S1, unofficial]. It is used in Unity 6 for volume occlusion and lighting of 3DGS content [S1, unofficial].

## Sources
[S1] arloopa/UnitySplats, https://github.com/arloopa/UnitySplats, fetched 2026-10-27 (unofficial)
