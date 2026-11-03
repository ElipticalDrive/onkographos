# Volumetric Cloud Raymarching

Also called: Cloud Raymarching, Cloud Volume Rendering, Ray Marching with Exponential Attenuation, Real-Time Volumetric Cloud Rendering, Volumetric Cloud Rendering

## Overview
Volumetric cloud raymarching is a technique for rendering volumetric clouds by tracing a ray through a 3D density field, accumulating color based on density, light scattering, and transmittance [S4, unofficial]. It differs from solid geometry ray marching by accumulating color and transmittance over a fixed number of steps or until transmittance drops below a threshold, rather than breaking on the first hit [S1, unofficial]. Unlike billboard or particle clouds, it represents true 3D density fields that interact with light realistically [S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position, camera ray direction, cloud volume bounds (min/max), light direction, 3D noise textures (base shape, Worley/cell noise, erosion), and rendering parameters (step size, extinction, scattering coefficients) [S1, unofficial][S2, unofficial][S4, unofficial].
*   **Outputs:** Final pixel color (RGB) and alpha/transmittance [S1, unofficial][S2, unofficial][S4, unofficial].

**Data Structures:**
*   **Cloud Volume Bounds:** `boundsMin` (float3), `boundsMax` (float3) defining the axis-aligned box containing the clouds [S2, unofficial].
*   **Noise Textures:** 3D textures (`Texture3D`) storing density values [S4, unofficial]. Common formats include `ARGBFloat` with dimensions like 128x128x128 [S4, unofficial]. Textures typically include base shape noise, Worley noise, and erosion noise [S4, unofficial].
*   **Uniforms/Constants:** Parameters vary by implementation. Source [S1, unofficial] uses `uCloudExtinction`, `uCloudLightStep`, `uCloudLightAtt`, `uCloudScatterMin`, `uCloudScatterMax`, `uCloudBrightness`, `uCloudStepMult`, `cloudCenter`, `cloudRadius`. Source [S2, unofficial] uses `boundsMin`, `boundsMax`, `cloud_densityThr`, `cloud_densityMul`, `cloud_offset`, `offsetSpeed`, `volume_march_steps`, `light_march_steps`, `detail_scale`, `detail_offset`, `detail_weight`, `frwrd_scatter`, `back_scatter`, `base_brightness`, `phase_factor`, `random_sample`, `optimized_march`, `lightAbsorptionThroughCloud`, `darknessThreshold`. Source [S4, unofficial] uses fixed values like `stepSize = 100.0`, `maxDistance = 30000.0`, `Density Threshold for Lighting = 0.01`, `Transmittance Attenuation Factor = 0.02`, `Transmittance Early Exit Threshold = 0.01`.

**Numbered Steps:**

1.  **Ray-Volume Intersection (Optional):**
    Calculate the intersection of the camera ray with the cloud volume [S1, unofficial].
    *   *Sphere Intersection:* If the volume is spherical, calculate `oc = ro - cloudCenter`, `b = dot(oc, rd)`, `c = dot(oc, oc) - cloudRadius * cloudRadius`, `disc = b*b - c` [S1, unofficial]. If `disc < 0`, return transparent. Entry `t0 = max(0.0, -b - sqrt(disc))`, Exit `t1 = min(maxDist, -b + sqrt(disc))` [S1, unofficial].
    *   *Box Intersection:* Compute `t0 = (boundsMin - rayOrigin) / rayDir` and `t1 = (boundsMax - rayOrigin) / rayDir` [S2, unofficial]. Determine `dstToBox = max(0.0f, max(max(tmin.x, tmin.y), tmin.z))` and `dstInsideObj = max(0.0f, min(tmax.x, min(tmax.y, tmax.z)) - dstToBox)` [S2, unofficial]. If `dstInsideObj <= 0`, the ray misses or starts inside; proceed with camera-origin marching [S2, unofficial].

2.  **Initialization:**
    Initialize step size, loop variable `t` (or `dstTravelled`), accumulated color `col` (or `result`), and transmittance `trans` (or `transmittance`) [S1, unofficial][S2, unofficial][S4, unofficial].
    *   *Step Size:* `step = (t1 - t0) / (96.0 * uCloudStepMult)` [S1, unofficial] or `stepSize = aDistance / volume_march_steps` [S2, unofficial] or fixed `100.0` [S4, unofficial].
    *   *Entry Point:* `entryPoint = rayOrigin + rayDir * dstToObj` [S2, unofficial] or start at `rayOrigin` [S4, unofficial].
    *   *Transmittance:* Initialize to `1.0` [S1, unofficial][S2, unofficial][S4, unofficial].

3.  **Marching Loop:**
    Iterate for a fixed number of steps (e.g., 96 [S1, unofficial], `volume_march_steps` [S2, unofficial], or 128 [S4, unofficial]) or until `transmittance < 0.01` [S1, unofficial][S2, unofficial][S4, unofficial].
    *   *Position:* `p = ro + rd * t` [S1, unofficial] or `rayPos = anEntryPoint + aRayDir * dstTravelled` [S2, unofficial][S4, unofficial].
    *   *Density Sampling:* Sample the density field at `p` [S4, unofficial].
        *   *Simple Method:* `density = SampleCloudDensity(p)` [S4, unofficial].
        *   *Advanced Method:* Sample low-res shape noise, apply threshold and multiplier, sample high-res detail noise (fBm), trim cloud using erosion and falloff functions [S2, unofficial].
            *   `uvw = (size * 0.5f + aPos) * baseScale * base_Scale` [S2, unofficial].
            *   `shapeNoise = shapeNoiseTex.Sample(uvw + cloud_offset * offsetSpeed).r` [S2, unofficial].
            *   `density = (shapeNoise - cloud_densityThr) * cloud_densityMul` [S2, unofficial].
            *   `detailFBM = SampleHighRes(uvw)` [S2, unofficial].
            *   `trimmedCloud = TrimCloud(aPos, density, detailFBM)` [S2, unofficial].
            *   `density = trimmedCloud * EdgeFallof(aPos) * HeightFallof(aPos)` [S2, unofficial].
    *   *Lighting Calculation:* If `density > 0` (or threshold like 0.001 [S1, unofficial], 0.01 [S4, unofficial]):
        *   *Shadow Ray:* Trace towards the light source to compute light attenuation [S1, unofficial].
            *   *Simple:* Loop 6 times, sampling density along light direction `L`, accumulating `lightSample += cloudDensityField(lp)`. `lightAtt = exp(-lightSample * uCloudLightAtt)` [S1, unofficial].
            *   *Advanced:* Call `LightMarch(rayPos, cosAngle)` which performs a box intersection with the cloud volume and accumulates optical depth using `MultipleOctaveScattering` [S2, unofficial].
        *   *Scatter Factor:* Implementations vary: `scatter = mix(uCloudScatterMin, uCloudScatterMax, lightAtt)` [S1, unofficial], compute phase function using Henyey-Greenstein [S2, unofficial], or use a direct light factor [S4, unofficial].
    *   *Color Accumulation:*
        *   `cloudColor = vec3(uCloudBrightness) * scatter` [S1, unofficial] or `lightEnergy += density * stepSize * transmittance * lightTransmittance * phaseVal` [S2, unofficial] or `result += density * light * transmittance` [S4, unofficial].
        *   *Transmittance Update:* `trans *= exp(-d * uCloudExtinction)` [S1, unofficial] or `transmittance *= exp(-density * stepSize * 0.02)` [S4, unofficial] or `transmittance *= BeersLaw(density * stepSize * lightAbsorptionThroughCloud)` [S2, unofficial].
    *   *Early Exit:* If `transmittance < 0.01`, break loop [S1, unofficial][S2, unofficial][S4, unofficial].
    *   *Advance:* `t += step` [S1, unofficial] or `dstTravelled += stepSize` [S2, unofficial][S4, unofficial].

4.  **Final Output:**
    Return `vec4(col, 1.0 - trans)` [S1, unofficial] or `float4(col, transmittance)` [S2, unofficial][S4, unofficial].

**Formulas:**
*   **Density Field:** `shape = smoothstep(1.2, 0.6, sphere) * flatBottom`, where `sphere = length(vec3(q.x, q.y * 0.55, q.z))`, `flatBottom = smoothstep(-0.2, 0.0, q.y)`, `q = (p - cloudCenter) / cloudRadius` [S1, unofficial].
*   **Beer's Law:** $T = e^{-aDist}$, where $aDist$ is the optical depth [S2, unofficial][S4, unofficial].
*   **Powder Function:** $P = 1 - e^{-2 \times aDist}$ [S2, unofficial].
*   **Henyey-Greenstein Phase Function:** $P(g, \cos\theta) = \frac{1 - g^2}{4 \pi (1 + g^2 - 2 g \cos\theta)^{1.5}}$ [S2, unofficial].
*   **Remap:** $v_{new} = minNew + (v_{old} - minOld) \times \frac{maxNew - minNew}{maxOld - minOld}$ [S2, unofficial].
*   **Density Composition:** `density = BaseNoise - Erosion` [S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_volumetric_clouds(ray_origin, ray_dir, sun_dir, bounds_min, bounds_max, max_distance, step_size, noise_textures):
    transmittance = 1.0
    accumulated_color = (0.0, 0.0, 0.0)
    pos = ray_origin

    for _ in range(int(max_distance / step_size)):
        # Check if inside cloud volume bounds
        if not (bounds_min.x <= pos.x <= bounds_max.x and
                bounds_min.y <= pos.y <= bounds_max.y and
                bounds_min.z <= pos.z <= bounds_max.z):
            break

        density = sample_cloud_density(pos, noise_textures)

        if density > 0.01:
            light = compute_sun_light(pos, sun_dir, noise_textures)
            accumulated_color = (
                accumulated_color[0] + density * light * transmittance,
                accumulated_color[1] + density * light * transmittance,
                accumulated_color[2] + density * light * transmittance
            )
            transmittance *= math.exp(-density * step_size * 0.02)

        if transmittance < 0.01:
            break

        pos = pos + ray_dir * step_size

    return (accumulated_color[0], accumulated_color[1], accumulated_color[2], 1.0 - transmittance)

def sample_cloud_density(pos, textures):
    uvw = pos * 0.0005
    base_noise = textures['base'].sample(uvw)[0]
    erosion = textures['erosion'].sample(uvw * 0.5)[0]
    density = base_noise - erosion * 0.6
    return max(0.0, min(1.0, density))

def compute_sun_light(pos, sun_dir, textures):
    light = 1.0
    step = 300.0
    current_pos = pos
    for _ in range(6):
        current_pos = current_pos + tuple(s * step for s in sun_dir)
        d = sample_cloud_density(current_pos, textures)
        light *= math.exp(-d * 0.1)
    return light
```

**Edge Cases and Pitfalls:**
*   **Light Marching Vector Bug:** In some implementations, adding a vector (`dirToLight * stepSize`) to a scalar (`dstTravelled`) and a scalar to a vector (`aPos`) is a logic error [S2, unofficial].
*   **Edge Falloff Bug:** Using `dstFromEdgeX` twice in the `min` function for edge weight ignores the Z-axis falloff [S2, unofficial].
*   **Precision:** Floating point precision issues may arise in large coordinate spaces; using local coordinates or normalized UVs helps [S4, unofficial].
*   **Degenerate Input:** If the ray does not intersect the volume, return transparent immediately [S1, unofficial][S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of raymarch steps ($O(N)$ where $N$ is `volume_march_steps` or fixed iterations like 96/128) [S1, unofficial][S4, unofficial]. Lighting calculations add a secondary loop ($O(M)$ where $M$ is `light_march_steps` or 6) [S1, unofficial][S4, unofficial].
*   **Memory Usage:** Requires storage for 3D noise textures. A typical implementation uses a 128x128x128 `ARGBFloat` texture [S4, unofficial].
*   **Measured Performance:** Rendering at quarter resolution with TAA upscale is common in AAA games to mitigate performance costs [S4, unofficial]. Typical step counts range from 64 to 128 [S4, unofficial].
*   **Strengths:** Realistic light interaction (volumetric shadows, subsurface scattering approximation), true 3D structure, smooth transitions [S4, unofficial].
*   **Limitations:** Computationally expensive due to multiple texture samples and lighting calculations per pixel [S4, unofficial].
*   **When to Use:** When realistic volumetric lighting and 3D cloud structure are required [S4, unofficial].
*   **When Not to Use:** When performance is critical and simple billboards or sprites are sufficient [S4, unofficial].

## Background
Volumetric cloud rendering has evolved from simple billboards to complex raymarched density fields [S4, unofficial]. Modern real-time implementations often use 3D noise textures (Perlin, Worley) combined with raymarching to approximate atmospheric scattering [S4, unofficial].

## Sources
[S1] Delyc/rayMarched-volumetric-rendering, https://github.com/Delyc/rayMarched-volumetric-rendering, fetched 2026-10-26 (unofficial)
[S2] OskarSchramm/Volumetric-Clouds, https://github.com/OskarSchramm/Volumetric-Clouds, fetched 2026-10-26 (unofficial)
[S4] Real‐Time Volumetric Cloud Rendering in Unity Using Raymarching, http://unityqueen.com/2026/08/29/real-time-volumetric-cloud-rendering-in-unity-using-raymarching/, fetched 2026-10-29 (unofficial)
