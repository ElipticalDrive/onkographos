# Volumetric Scattering via Ray Marching (Single Light)

Also called: Exponential Fog Light Shafts, Ray Marching Light Volume Integration, Ray Marching with Spot Factor, Tomas Ohlsson's volumetric shader, Volumetric.frag method

## Overview
This method simulates volumetric light shafts for multiple spot lights by ray marching from the camera through the scene volume, integrating scattering contributions along the view ray [S1, unofficial]. It operates as a post-processing compute shader pass that explicitly calculates light attenuation, spot cone factors, and phase function scattering at each sample point [S1, unofficial].

## How it works
The method reconstructs world-space positions from depth, then marches along the view vector to accumulate scattered light contributions [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `inputTexture`: Color buffer [S1, unofficial].
    *   `depthTexture`: Depth buffer [S1, unofficial].
    *   `gInvProj`: Inverse projection matrix [S1, unofficial].
    *   `gInvView`: Inverse view matrix [S1, unofficial].
    *   `gCameraPos`: Camera position in world space [S1, unofficial].
    *   `gLights`: Array of light structures containing `Position`, `Direction`, `Range`, `Color`, `VolumetricStrength`, `outerCosine`, `innerCosine` [S1, unofficial].
    *   Constants: `absorptionTau`, `absorptionColor`, `scatteringSamples`, `scatteringTau`, `scatteringZFar`, `scatteringColor` [S1, unofficial].
*   **Outputs**:
    *   `outputTexture`: Accumulated volumetric scattering color [S1, unofficial].

### Data Structures
**Constant Buffer 'VolumetricCB' Layout** [S1, unofficial]:
*   `c0.x`: `absorptionTau` (float)
*   `c0.y`: `absorptionColor` (float3)
*   `c1.x`: `scatteringSamples` (int)
*   `c1.y`: `scatteringTau` (float)
*   `c1.z`: `scatteringZFar` (float)
*   `c2`: `scatteringColor` (float3)
*   `c3-c6`: `gInvProj` (matrix)
*   `c7-c10`: `gInvView` (matrix)
*   `c11`: `gCameraPos` (float3)
*   `c12+`: `gLights` (array of Light structs)

**Light Struct Fields** [S1, unofficial]:
*   `Position`: float3
*   `Direction`: float3
*   `Range`: float
*   `Color`: float3
*   `VolumetricStrength`: float
*   `outerCosine`: float
*   `innerCosine`: float
*   `SpotPower`: float
*   `Type`: int

### Numbered Steps
1.  **Reconstruct World Position**: Get pixel dimensions, convert pixel coordinates to NDC (-1 to 1), multiply by inverse projection matrix, divide by w, multiply by inverse view matrix [S1, unofficial].
2.  **Initialize Ray March**: Calculate vector from camera to fragment [S1, unofficial]. If distance > scatteringZFar, clamp length to scatteringZFar [S1, unofficial]. Calculate deltaStep = camToFrag / (scatteringSamples + 1) [S1, unofficial]. Offset start position by random value: x = gCameraPos + deltaStep * random(worldPosition.xy + worldPosition.z) [S1, unofficial].
3.  **Scattering Loop**: For i from 0 to scatteringSamples-1: Calculate light vector L = x - light.Position [S1, unofficial]. Calculate light distance dist = |L| [S1, unofficial]. Calculate solid angle factor omega = 4 * PI * dist^2 [S1, unofficial]. Calculate incoming radiance Lin = T_abs(dist) * visibility * light.Diffuse * light.SpotPower / omega [S1, unofficial]. Calculate scattered radiance Li = Lin * scatteringTau * scatteringColor * phaseFunction(normalize(L), normalize(gCameraPos - worldPosition)) [S1, unofficial]. Add to result: result += Li * T_abs(|x - gCameraPos|) * |deltaStep| [S1, unofficial]. Advance x += deltaStep [S1, unofficial].
4.  **Main Dispatch**: For each pixel, reconstruct world position [S1, unofficial]. For each light, if Type == SPOT_LIGHT and distance to light < FalloffEnd, add volumetricScattering result to output [S1, unofficial].

### Formulas
**Absorption Transmittance** [S1, unofficial]:
$$ T_{abs}(dist) = C_{abs} \times e^{-dist \times (\tau_{abs} + \tau_{scat})} $$
Where:
*   $C_{abs}$: `absorptionColor`
*   $\tau_{abs}$: `absorptionTau`
*   $\tau_{scat}$: `scatteringTau`

**Phase Function** [S1, unofficial]:
$$ P(\theta) = \frac{0.5 + 16.5 \times x^{32}}{4\pi} $$
Where:
*   $x = \frac{1 + \cos\theta}{2}$
*   $\cos\theta = \frac{\mathbf{inDir} \cdot \mathbf{outDir}}{|\mathbf{inDir}| |\mathbf{outDir}|}$ [S1, unofficial].

### Parameters
*   `absorptionTau`: float, typical value $-0.061$ [S1, unofficial]. Controls absorption coefficient.
*   `scatteringTau`: float, typical value $0.059$ [S1, unofficial]. Controls scattering coefficient.
*   `scatteringSamples`: int, typical value $16$ [S1, unofficial]. Number of samples per view ray.
*   `scatteringZFar`: float. Maximum distance for ray marching.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

SPOT_LIGHT = 1

def length(v):
    return math.sqrt(sum(c * c for c in v))

def dot(a, b):
    return sum(x * y for x, y in zip(a, b))

def normalize(v):
    l = length(v)
    if l == 0: return [0.0, 0.0, 0.0]
    return [c / l for c in v]

def random(co):
    return math.fmod(math.sin(dot(co, [12.9898, 78.233])) * 43758.5453123, 1.0)

def absorption_transmittance(dist, absorption_color, absorption_tau, scattering_tau):
    return [c * math.exp(-dist * (absorption_tau + scattering_tau)) for c in absorption_color]

def phase_function(in_dir, out_dir):
    cos_angle = dot(in_dir, out_dir) / (length(in_dir) * length(out_dir))
    x = (1.0 + cos_angle) / 2.0
    return (0.5 + 16.5 * (x ** 32)) / (4.0 * math.pi)

def volumetric_scattering(world_position, camera_pos, light, scattering_samples, scattering_tau, scattering_zfar, scattering_color, absorption_tau, absorption_color):
    cam_to_frag = [world_position[i] - camera_pos[i] for i in range(3)]
    dist_cf = length(cam_to_frag)

    if dist_cf > scattering_zfar:
        cam_to_frag = normalize(cam_to_frag)
        cam_to_frag = [c * scattering_zfar for c in cam_to_frag]

    delta_step = [c / (scattering_samples + 1) for c in cam_to_frag]
    frag_to_cam_norm = normalize([camera_pos[i] - world_position[i] for i in range(3)])

    # HLSL quirk: worldPosition.xy + worldPosition.z adds scalar to vector components
    rand_val = random([world_position[0], world_position[1] + world_position[2]])
    x = [camera_pos[i] + delta_step[i] * rand_val for i in range(3)]

    result = [0.0, 0.0, 0.0]

    for i in range(scattering_samples):
        light_to_x = [x[i] - light['Position'][i] for i in range(3)]
        light_dist = length(light_to_x)

        omega = 4.0 * math.pi * light_dist * light_dist

        t_abs_light = absorption_transmittance(light_dist, absorption_color, absorption_tau, scattering_tau)

        visibility = 1.0  # [G] Simplified constant; sources note shadow maps should be used for correctness

        lin = [t_abs_light[i] * visibility * light['Diffuse'][i] * light['SpotPower'] / omega for i in range(3)]

        p_func = phase_function(normalize(light_to_x), frag_to_cam_norm)

        li = [lin[i] * scattering_tau * scattering_color[i] * p_func for i in range(3)]

        dist_x_cam = length([x[i] - camera_pos[i] for i in range(3)])
        t_abs_cam = absorption_transmittance(dist_x_cam, absorption_color, absorption_tau, scattering_tau)

        delta_len = length(delta_step)
        result = [result[i] + li[i] * t_abs_cam[i] * delta_len for i in range(3)]

        x = [x[i] + delta_step[i] for i in range(3)]

    return result

def compute_shader(pixel, depth, input_texture, output_texture, inv_proj, inv_view, camera_pos, lights, absorption_tau, absorption_color, scattering_samples, scattering_tau, scattering_zfar, scattering_color):
    width, height = input_texture.dimensions()
    u, v = pixel
    x_ndc = (u / width) * 2.0 - 1.0
    y_ndc = (v / height) * -2.0 + 1.0
    z_ndc = depth

    ndc = [x_ndc, y_ndc, z_ndc, 1.0]
    p_view = [sum(ndc[j] * inv_proj[j][i] for j in range(4)) for i in range(4)]
    p_view = [p_view[i] / p_view[3] for i in range(3)]
    world_pos = [sum(p_view[j] * inv_view[j][i] for j in range(4)) for i in range(3)]

    volumetric_color = [0.0, 0.0, 0.0]

    for light in lights:
        if light['Type'] == SPOT_LIGHT:
            dist_light = length([light['Position'][i] - world_pos[i] for i in range(3)])
            if dist_light < light['FalloffEnd']:
                vol = volumetric_scattering(world_pos, camera_pos, light, scattering_samples, scattering_tau, scattering_zfar, scattering_color, absorption_tau, absorption_color)
                volumetric_color = [volumetric_color[i] + vol[i] for i in range(3)]

    output_texture[pixel] = [volumetric_color[i] + input_texture[pixel][i] for i in range(3)]
```

### Edge Cases and Pitfalls
*   **Visibility**: Setting visibility to a constant $1.0$ ignores shadow maps and results in undesired output (light shafts appearing through occluders) [S1, unofficial]. Correct visibility calculation is critical [S1, unofficial].
*   **Light Direction Transformation**: Multiplying light direction by the view matrix causes incorrect volume direction for vehicle headlights, while not multiplying causes incorrect direction for street lamps, and the correct transformation remains unclear in the source text [S1, unofficial].
*   **Precision**: Using `float` for depth reconstruction can lead to precision issues at large distances.
*   **Boundary**: Ray marching is clamped to `scatteringZFar` [S1, unofficial]. Samples beyond this distance are ignored.

## Subtopics

### Scan-line Volume Rendering
A scan-line based method that deals with shadow volumes to take into account the glow in haze, computing the radiance reaching the eye by adding the contributions of single scattering of light in illumination volumes [S2]. It considers two scenarios: a light source at infinity illuminating a fog layer with constant or layered density with any phase function, and a point light source with a medium of constant density and isotropic scattering [S2].

## Operating characteristics
Excessive frame drops occur when calculating visibility for 32 lights using shadow maps [S1, unofficial]. Not given in the sources: exact time/memory complexity formulas, measured performance metrics, or scaling laws.

## Background
This method is derived from post-processing techniques for simulating volumetric scattering in real-time graphics, referencing Tomas Ohlsson's volumetric shader (`volumetric.frag`) and concepts from GPU Gems 3, Chapter 13 [S1, unofficial]. It was developed as a compute shader approach for DirectX 12 to handle multiple spot lights efficiently [S1, unofficial].

## Sources
[S1] DX12) Trying to Implement Volumetric Scattering for multiple Spot Light, but It's not going well, https://stackoverflow.com/questions/71985186/dx12-trying-to-implement-volumetric-scattering-for-multiple-spot-light-but-it, fetched 2026-10-27 (unofficial)
[S2] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
