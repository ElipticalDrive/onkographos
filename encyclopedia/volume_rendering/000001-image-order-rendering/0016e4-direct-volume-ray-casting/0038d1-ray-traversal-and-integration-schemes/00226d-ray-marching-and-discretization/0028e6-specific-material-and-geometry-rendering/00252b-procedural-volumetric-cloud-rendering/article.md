# Procedural Volumetric Cloud Rendering

Also called: Nubis³, Nubis Cubed, Nubis Evolved, Volumetric Cloud Renderer, Volumetric Cloud Rendering, Real-time Volumetric Cloud Rendering

## Overview

Procedural volumetric cloud rendering is a real-time technique that ray-marches a spherical cloud layer built from tileable 3D noise fields, lighting it with multiple-scattering approximations and Mie phase functions [S3, S4, unofficial]. It solves the problem of rendering physically plausible, large-scale cloudscapes with dynamic lighting and shadows at interactive frame rates [S2, S3, S4, unofficial]. What sets it apart from [Mr-Voxel Volume Rendering](3874-mr-voxel-volume-rendering.md) is the use of procedural noise for density generation rather than pre-computed voxel grids, and from [Ray-marched Volumetric Fog](4248-ray-marched-volumetric-fog.md) by employing specific cloud physics (Mie scattering, multiple scattering approximations, and vertical density profiles) rather than generic volumetric fog [S1, S3, S4, unofficial].

## How it works

### Inputs and Outputs

**Inputs:**
*   **Camera:** Position, view-projection matrix, viewport dimensions [S3, S4, unofficial].
*   **Lighting:** Sun direction (spherical coordinates converted to Cartesian), sun color, sun intensity [S2, S3, S4, unofficial].
*   **Cloud Parameters:** Base height, thickness, coverage, density multiplier, wind speed/vector, anisotropy, extinction, ambient intensity, erosion amount [S3, unofficial].
*   **Textures:**
    *   `uShape`: 128³ sampler3D (Perlin fbm dilated by Worley, plus three Worley octaves) [S3, unofficial].
    *   `uDetail`: 32³ sampler3D (Worley detail) [S3, unofficial].
    *   `uWeather`: 512² sampler2D (coverage and cloud type per region) [S3, unofficial].
    *   `solidDepth`: 2D texture for solid geometry occlusion [S2, unofficial].
    *   `solidColor`: 2D texture for solid geometry color [S2, unofficial].
*   **Constants:** `EARTH_RADIUS` (6,371,000 m), `SHAPE_SCALE` (1/9000 m/tile), `DETAIL_SCALE` (1/900), `WEATHER_SCALE` (1/48,000) [S3, unofficial].

**Outputs:**
*   `outColor`: `vec4` with RGB as in-scattered radiance, A as transmittance [S3, unofficial].
*   `outDepth`: `float` representing metres along the ray (0 if no cloud) [S3, unofficial].

### Data Structures

*   **Vertex Input:** `struct VertexInput { vec4 vertex; vec2 uv; }` [S4, unofficial].
*   **Vertex Output:** `struct VertexOutput { vec4 screenPosition; vec2 uv; vec3 rayDirection; }` [S4, unofficial].
*   **Cloud State:** Per-view ownership of persistent textures (LUTs, cloud field, shadow map, history) [S1, unofficial].

### Steps

1.  **Vertex Shader Setup:**
    *   Convert vertex position to clip space [S4, unofficial].
    *   Determine ray direction based on UV quadrant to cover the frustum corners:
        *   If `uv.x < 0.5 && uv.y < 0.5`: use `_FrustumCornerBL` [S4, unofficial].
        *   If `uv.x < 0.5 && uv.y >= 0.5`: use `_FrustumCornerTL` [S4, unofficial].
        *   If `uv.x >= 0.5 && uv.y < 0.5`: use `_FrustumCornerBR` [S4, unofficial].
        *   Else: use `_FrustumCornerTR` [S4, unofficial].
    *   Transform corners to world space using camera rotation only [S4, unofficial].

2.  **Ray Setup:**
    *   Normalize `rayDirWorldspace` to get `rayDirWorld` [S2, unofficial].
    *   Intersect ray with the spherical cloud shell:
        *   Calculate `oc = ro - vec3(0.0, -EARTH_RADIUS, 0.0)` [S3, unofficial].
        *   `rCam = length(oc)` [S3, unofficial].
        *   `b = dot(oc, rd)` [S3, unofficial].
        *   `c = dot(oc, oc)` [S3, unofficial].
        *   Compute `hOuter = b * b - (c - rOuter * rOuter)` [S3, unofficial]. If `hOuter < 0.0`, return false [S3, unofficial].
        *   Compute `sOuter = sqrt(hOuter)` [S3, unofficial].
        *   `tOuterNear = -b - sOuter` [S3, unofficial].
        *   `tOuterFar = -b + sOuter` [S3, unofficial].
        *   Compute `hInner = b * b - (c - rInner * rInner)` [S3, unofficial]. If `hInner > 0.0`, `sInner = sqrt(hInner)`, else `sInner = 0.0` [S3, unofficial].
        *   Intersection interval is `[tOuterNear, tOuterFar]` intersected with inner shell bounds if applicable [S3, unofficial].
    *   Intersect ray with solid geometry box (if present) to get `tHitSolid` [S2, unofficial].
    *   Clamp near intersection `tHit.x` to `max(0, tHit.x)` and far intersection `tHit.y` to `min(tHit.y, tHitSolid)` [S2, unofficial].
    *   Calculate start point `pointWorld = rayOrigWorld + tHit.x * rayDirWorld` [S2, unofficial].

3.  **Density Sampling:**
    *   Transform position using high-res scaling: `hiResPosition = outerProduct(position, hiResS) + outerProduct(hiResT, vec4(1.f))` where `hiResT = .1f * hiResNoiseTranslate` and `hiResS = .1f * hiResNoiseScaling` [S2, unofficial].
    *   Sample 4 textures from `volumeHighRes` at `hiResPosition[0]` (R), `[1]` (G), `[2]` (B), `[3]` (A) [S2, unofficial].
    *   Compute `hiResDensity = dot(hiResNoise, normalizeL1(hiResChannelWeights))` [S2, unofficial].
    *   If `invertDensity`, `hiResDensity = 1.f - hiResDensity` [S2, unofficial].
    *   Apply falloff: `falloff = yFalloff(position) * xzFalloff(position)` [S2, unofficial].
        *   `yFalloff`: `distY = min(Y_FALLOFF_DIST, min(position.y - ymin, ymax - position.y))`. Return `distY / Y_FALLOFF_DIST` [S2, unofficial].
        *   `xzFalloff`: `distX = min(XZ_FALLOFF_DIST, min(position.x - xmin, xmax - position.x))`, `distZ = min(XZ_FALLOFF_DIST, min(position.z - zmin, zmax - position.z))`. Return `min(distX, distZ) / XZ_FALLOFF_DIST` [S2, unofficial].
    *   `hiResDensity *= falloff` [S2, unofficial].
    *   Add offset: `hiResDensityWithOffset = hiResDensity + hiResDensityOffset` [S2, unofficial].
    *   If `hiResDensityWithOffset <= 0.f`, return 0.f [S2, unofficial].
    *   Sample low-res: `loResPosition = position * loResNoiseScaling * .1f + loResNoiseTranslate` [S2, unofficial].
    *   `loResNoise = texture(volumeLowRes, loResPosition)` [S2, unofficial].
    *   `loResDensity = dot(loResNoise, normalizeL1(loResChannelWeights))` [S2, unofficial].
    *   Invert: `loResDensity = 1.f - loResDensity` [S2, unofficial].
    *   Compute erosion weight: `erosionWeight = getErosionWeightQuntic(hiResDensity)` where `getErosionWeightQuntic(d) = pow((1.f - d), 6)` [S2, unofficial].
    *   Final density: `density = hiResDensityWithOffset - erosionWeight * loResDensityWeight * loResDensity` [S2, unofficial].
    *   Return `max(density * densityMult * 5.f, 0.f)` [S2, unofficial].

4.  **Lighting Optical Depth:**
    *   Each march step takes six samples toward the sun to get optical depth [S3, unofficial].
    *   Intersect ray with box `tHit = intersectBox(rayOrig, rayDir)` [S2, unofficial].
    *   `tFar = max(0.f, tHit.y)` [S2, unofficial].
    *   `dt = tFar / numStepsRecursive` [S2, unofficial].
    *   `ds = rayDir * dt` [S2, unofficial].
    *   Initialize `tau = 0.f`, `pointWorld = rayOrig` [S2, unofficial].
    *   Loop `t` from 0 to `tFar` with step `dt`:
        *   `density = sampleDensity(pointWorld)` [S2, unofficial].
        *   `tau -= density` [S2, unofficial].
        *   `pointWorld += ds` [S2, unofficial].
    *   After loop: `tau *= (cloudLightAbsorptionMult * dt)` [S2, unofficial].
    *   `lightTransmittance = exp(tau)` [S2, unofficial].

5.  **Ray Marching Integration:**
    *   Step size is derived from the ray's span through the layer [S3, unofficial].
    *   Steps are jittered per pixel with interleaved gradient noise that changes each frame [S3, unofficial].
    *   March termination: The march stops once transmittance drops below 0.5% [S3, unofficial].
    *   In-scattering is integrated analytically over each step so the result does not depend on step size [S3, unofficial].
    *   Lighting evaluation per step:
        *   Calculate sun direction `sunDirSpherical` using `dirSph2Cart` from sun latitude/longitude [S2, unofficial].
        *   Compute sun position `sunPos = SUN_RADIUS * sunDirSpherical` [S2, unofficial].
        *   Calculate light direction `dirLight = normalize(sunPos - pointWorld)` [S2, unofficial].
        *   Calculate cosine of angle between ray and light `cosRayLightAngle = dot(rayDirWorld, dirLight)` [S2, unofficial].
        *   Compute phase value `phaseVal = phase(cosRayLightAngle)` [S2, unofficial].
        *   Get sun color `sunColor = getSunColor(sunLongitudeRadians)` [S2, unofficial].
        *   Evaluate four-octave multiple-scattering approximation (attenuated extinction, contribution, phase eccentricity per octave) with a two-lobe Henyey-Greenstein phase function [S3, unofficial].
        *   Apply powder term to darken back-lit edges [S3, unofficial].
        *   Apply ambient term fading from blue at top (`vec3(0.62, 0.74, 0.95)`) to grey at bottom (`vec3(0.36, 0.42, 0.52)`) [S3, unofficial].
        *   Sun/sky color shift warm as sun nears horizon [S3, unofficial].

6.  **Temporal Reprojection (Optional but common in real-time implementations):**
    *   Render at reduced resolution (e.g., half or quarter) with sub-pixel jitter from a Halton sequence [S3, unofficial].
    *   Resolve pass reconstructs world position from cloud depth, projects through previous frame's view-projection matrix, and blends history sample at 92% (blend factor 0.08f) [S3, unofficial].
    *   History is clamped to 3×3 neighbourhood to prevent ghosting from disocclusions [S3, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def normalizeL1(v):
    return v / dot(v, vec4(1.0))

def calculate_y_falloff(pos, params):
    ymin = -0.5 * params.volume_scaling.y + params.volume_translate.y
    ymax = ymin + params.volume_scaling.y
    dist_y = min(params.y_falloff_dist, min(pos.y - ymin, ymax - pos.y))
    return dist_y / params.y_falloff_dist

def calculate_xz_falloff(pos, params):
    xmin = -0.5 * params.volume_scaling.x + params.volume_translate.x
    xmax = xmin + params.volume_scaling.x
    zmin = -0.5 * params.volume_scaling.z + params.volume_translate.z
    zmax = zmin + params.volume_scaling.z
    dist_x = min(params.xz_falloff_dist, min(pos.x - xmin, xmax - pos.x))
    dist_z = min(params.xz_falloff_dist, min(pos.z - zmin, zmax - pos.z))
    return min(dist_x, dist_z) / params.xz_falloff_dist

def sample_cloud_density(pos, textures, params):
    hi_res_pos = outer_product(pos, params.hi_res_s) + outer_product(params.hi_res_t, vec4(1.0))
    hi_res_noise = texture(textures.volume_high_res, hi_res_pos)
    hi_res_density = dot(hi_res_noise, normalizeL1(params.hi_res_channel_weights))
    if params.invert_density:
        hi_res_density = 1.0 - hi_res_density
    falloff = calculate_y_falloff(pos, params) * calculate_xz_falloff(pos, params)
    hi_res_density *= falloff
    hi_res_density_with_offset = hi_res_density + params.hi_res_density_offset
    if hi_res_density_with_offset <= 0.0:
        return 0.0
    lo_res_pos = pos * params.lo_res_noise_scaling * 0.1 + params.lo_res_noise_translate
    lo_res_noise = texture(textures.volume_low_res, lo_res_pos)
    lo_res_density = dot(lo_res_noise, normalizeL1(params.lo_res_channel_weights))
    lo_res_density = 1.0 - lo_res_density
    erosion_weight = pow(1.0 - hi_res_density, 6)
    density = hi_res_density_with_offset - erosion_weight * params.lo_res_density_weight * lo_res_density
    return max(density * params.density_mult * 5.0, 0.0)

def henyey_greenstein(cos_theta, g):
    g2 = g * g
    return (1.0 - g2) / (4.0 * 3.14159 * pow(1.0 + g2 - 2.0 * g * cos_theta, 1.5))

def calculate_sun_transmittance(point, ray_dir, density, textures, params):
    tau = 0.0
    ds = ray_dir * (params.t_far / 6.0)
    current_point = point
    for _ in range(6):
        d = sample_cloud_density(current_point, textures, params)
        tau -= d
        current_point += ds
    tau *= params.cloud_light_absorption_mult * (params.t_far / 6.0)
    return exp(tau)

def calculate_multiple_scattering(point, density, sun_transmittance, phase, params):
    # Four-octave approximation after Wrenninge and Hillaire [S3, unofficial]
    scattering = vec3(0.0)
    for octave in range(4):
        attenuation = exp(-density * params.extinction * (octave + 1))
        contribution = params.sun_intensity * sun_transmittance * attenuation
        eccentricity = 1.0 - (octave * 0.2)
        scattering += contribution * phase * eccentricity
    return scattering

def ray_march_clouds(ray_orig, ray_dir, cloud_params, textures, frame):
    oc = ray_orig - vec3(0.0, -6371000.0, 0.0)
    b = dot(oc, ray_dir)
    c = dot(oc, oc)
    h_outer = b * b - (c - cloud_params.r_outer ** 2)
    if h_outer < 0.0:
        return vec4(0.0), 0.0
    s_outer = sqrt(h_outer)
    t_outer_near = -b - s_outer
    t_outer_far = -b + s_outer
    h_inner = b * b - (c - cloud_params.r_inner ** 2)
    s_inner = sqrt(h_inner) if h_inner > 0.0 else 0.0
    t_start = max(0.0, t_outer_near)
    t_end = t_outer_far
    ray_length = t_end - t_start
    step_size = ray_length / cloud_params.max_steps
    jitter = (halton(frame % 16 + 1, 2) - 0.5) * 2.0 / cloud_params.cloud_width
    step_size *= (1.0 + jitter)
    point = ray_orig + ray_dir * t_start
    transmittance = 1.0
    radiance = vec3(0.0)
    depth_accum = 0.0
    transmittance_weighted_depth = 0.0
    for i in range(cloud_params.max_steps):
        if transmittance < 0.005:
            break
        density = sample_cloud_density(point, textures, cloud_params)
        if density > 0.0:
            sun_transmittance = calculate_sun_transmittance(point, ray_dir, density, textures, cloud_params)
            sun_dir = normalize(vec3(0.0, 100.0, 0.0) - point)
            cos_angle = dot(normalize(ray_dir), normalize(sun_dir))
            phase = henyey_greenstein(cos_angle, cloud_params.anisotropy)
            scattering = calculate_multiple_scattering(point, density, sun_transmittance, phase, cloud_params)
            height_factor = (point.y - cloud_params.base_height) / cloud_params.thickness
            ambient = mix(vec3(0.36, 0.42, 0.52), vec3(0.62, 0.74, 0.95), height_factor)
            step_radiance = scattering * (1.0 - exp(-density * step_size))
            radiance += step_radiance * transmittance
            transmittance *= exp(-density * step_size)
            depth_accum += step_size
            transmittance_weighted_depth += step_size * transmittance
        point += ray_dir * step_size
    mean_depth = transmittance_weighted_depth / (1.0 - transmittance) if transmittance < 0.999 else 0.0
    return radiance, transmittance, mean_depth
```

### Parameters

*   `shape_noise_scale`: Default 6.4 km. Sets cloud size. At this scale, cloud size measures 590-760 m on GPU noise dumps [S1, unofficial].
*   `coverage`: Scales weather coverage. With physical profiles, coverage times the vertical profile is the cloud fraction [S1, unofficial].
*   `uShape`: 128³ sampler3D for shape noise [S3, unofficial].
*   `uDetail`: 32³ sampler3D for detail noise [S3, unofficial].
*   `uWeather`: 512² sampler2D for weather map [S3, unofficial].
*   `absorptionCoefficient`: Controls light absorption via Beer-Lambert law; default 1.0, range [0.0, 5.0] [S4, unofficial].
*   `densityThreshold`: Minimum noise value to contribute to cloud density; range [0.0, 1.0] [S4, unofficial].
*   `scatterFactor`: Controls forward light scattering strength; 0 is weakest, 1 is strongest; range [0.0, 1.0] [S4, unofficial].
*   `cloudScale`: Controls zoom level into noise texture; range [0.001, 0.05] [S4, unofficial].
*   `octaveCount`: Number of noise layers combined for Fractal Brownian Motion; range [1, 4] [S4, unofficial].
*   `windSpeed`: Controls speed of cloud movement; range [0.0, 10.0] [S4, unofficial].
*   `uMaxSteps`: Maximum marching steps, default 128 [S3, unofficial].
*   `s.scale`: Resolution scale factor for temporal reprojection, default 0.5 [S3, unofficial].
*   `EARLY_STOP_THRESHOLD`: 1e-2f. Threshold for early stopping [S2, unofficial].
*   `EARLY_STOP_LOG_THRESHOLD`: -4.6f. Log transmittance threshold for early stopping [S2, unofficial].
*   `MIN_NUM_FINE_STEPS`: 16. Minimum number of fine steps for adaptive ray marching [S2, unofficial].
*   `COARSE_STEPSIZE_MULTIPLIER`: 4.f. Multiplier for coarse step size relative to fine step size [S2, unofficial].
*   `MAX_NUM_MISSED_STEPS`: 5. Maximum number of missed steps in adaptive ray marching [S2, unofficial].
*   `STEPSIZE_FINE`: 0.02f. Fixed step size for fine ray marching [S2, unofficial].
*   `MAX_SUN_INTENSITY`: 4.f. Maximum sun intensity [S2, unofficial].
*   `XZ_FALLOFF_DIST`: 1.f. Distance for XZ falloff calculation [S2, unofficial].
*   `Y_FALLOFF_DIST`: 1.f. Distance for Y falloff calculation [S2, unofficial].
*   `SUN_RADIUS`: 100.f. Radius of the sun in world units [S2, unofficial].
*   `AMBIENT_TOP`: `vec3(0.62, 0.74, 0.95)` ambient color at top of layer [S3, unofficial].
*   `AMBIENT_BOTTOM`: `vec3(0.36, 0.42, 0.52)` ambient color at bottom of layer [S3, unofficial].
*   `uBlend`: Blending factor in temporal resolve pass, set to 0.08f [S3, unofficial].

### Edge Cases and Pitfalls

*   **Texture History Validity:** History validity is checked by texture identity. A new view, reallocation, or resize starts from an invalid history [S1, unofficial].
*   **Shadow Map Basis Error:** The shadow map is traced along the view's first directional light. If every consumer looked it up for each light in that light's own basis, a second, nonparallel sun read the first one's optical depths as a displaced, wrongly oriented shadow [S1, unofficial].
*   **Viewport Depth Mismatch:** Cloud raymarch and half-res haze prepass sized their pixel lattice from the depth texture (covering the whole render target) while their targets cover the viewport, reading depth without the viewport origin. An inset camera traced the wrong part of the view and the wrong depths. Both must use `view.viewport` for the lattice and offset depth reads by its origin [S1, unofficial].
*   **Light Sorting Flag Order:** The sort key `(volumetric, shadow_maps_enabled, entity)` is ascending, so lights without these came first. A plain directional light then took index 0, the volumetric sun lost its VOLUMETRIC flag. Reverse the two flags; the atmosphere's CPU mirror of the order does the same [S1, unofficial].
*   **Reprojection Depth Compromise:** Reprojection uses the mean cloud depth per pixel. Where a thin near cloud overlaps a far one, that depth is a compromise and the history clamp does the rest [S3, unofficial].
*   **Sky Model Limitation:** Sky is an analytic gradient, not an atmosphere model, so the sun's colour at low elevation is hand-tuned [S3, unofficial].
*   **Multiple Scattering Approximation:** The multiple-scattering octaves are an approximation; the lit side of thick clouds is somewhat flatter than in a path-traced reference [S3, unofficial].

## Operating characteristics

*   **Time Complexity:** O(N * M) where N is the number of pixels and M is the number of marching steps per pixel [G]. With temporal reprojection, the effective cost is reduced by the resolution scale factor squared [S3, unofficial].
*   **Memory Complexity:** O(T) where T is the size of the noise textures (128³ + 32³ + 512²) and history buffers [G]. Per-view ownership of textures increases memory usage with multiple views [S1, unofficial].
*   **Measured Performance:** On Mesa llvmpipe (CPU rasterizer) at 640×360: full-res, 128 steps, no temporal costs 927 ms total; half-res, 128 steps, temporal costs 266 ms total (29% of full-res) [S3, unofficial].
*   **Scaling:** Temporal reprojection allows rendering at half or quarter resolution with minimal quality loss, converging to full-resolution over roughly a dozen frames [S3, unofficial].
*   **Strengths:** Real-time performance, physically plausible lighting, dynamic shadows, large-scale cloudscapes [G].
*   **Limitations:** Approximation in multiple scattering, hand-tuned sky color, single cloud layer support in some implementations, history validity issues with view changes [S1, S3, unofficial].
*   **When to use:** Real-time games, simulations requiring dynamic cloud lighting and shadows [G].
*   **When not to use:** Offline rendering requiring path-traced accuracy, scenes with multiple distinct cloud layers requiring separate handling (unless extended) [G].

## Background

Procedural volumetric cloud rendering originated from techniques presented in SIGGRAPH talks by Andrew Schneider (Horizon: Zero Dawn, 2015) and Sebastian Hillaire (Frostbite, 2016; Unreal Engine, 2020) [S1, S2, S3, unofficial]. The Nubis series of techniques (Nubis Evolved 2022, Nubis³ 2023) by Andrew Schneider further refined the procedural noise shaping and multiple scattering approximations [S1, unofficial]. The method is implemented in engines such as Bevy [S1, unofficial] and Unity [S4, unofficial].

## Sources

[S1] WIP: Nubis-based volumetric clouds, https://github.com/mate-h/bevy/pull/26, fetched 2026-10-27 (unofficial)
[S2] Depersonalizc/volumetric-clouds, https://github.com/Depersonalizc/volumetric-clouds, fetched 2026-10-27 (unofficial)
[S3] harkawataryan/Volumetric-Cloud-Renderer, https://github.com/harkawataryan/Volumetric-Cloud-Renderer, fetched 2026-10-27 (unofficial)
[S4] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
