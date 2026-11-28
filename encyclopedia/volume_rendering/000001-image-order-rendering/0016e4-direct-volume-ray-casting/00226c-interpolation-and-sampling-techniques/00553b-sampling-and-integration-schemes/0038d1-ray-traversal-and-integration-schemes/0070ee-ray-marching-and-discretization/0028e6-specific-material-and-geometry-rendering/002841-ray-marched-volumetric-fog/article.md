# Ray-marched Volumetric Fog

Also called: Fog Filter, Fog Scattering, Forward ray-marching fog, Full-screen volumetric fog, Volumetric Fog (CRYENGINE 3), Voxel-based Volumetric Fog

## Overview
Ray-marched volumetric fog simulates aerial perspective and light shafts by treating fog as a 3D volume of participating media traced via forward ray-marching from the camera to the scene [S3, unofficial]. It solves the problem of rendering dynamic, light-interacting fog that absorbs and scatters light, distinguishing itself from backward ray-marching by allowing early-out optimizations when transmittance becomes negligible [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position, per-pixel UV coordinates, depth buffer, light position and intensity $\Phi$, fog density $\tau$, scattering coefficient $a$, step size $\Delta l$, maximum ray distance `zfar`, and phase function asymmetry parameter $g$ [S3, unofficial].
*   **Outputs:** Final radiance $L$ per pixel [S3, unofficial].

**Data Structures:**
*   **Ray:** Defined by origin (camera position) and direction (view vector) [S3, unofficial].
*   **Depth Buffer:** 2D texture mapping UV coordinates to depth values $z$ [S3, unofficial].
*   **Light Source:** Point light defined by position and intensity $\Phi$ [S3, unofficial].

**Numbered Steps:**
1.  **Ray Initialization:** For each pixel, construct a ray originating at the camera position [S3, unofficial].
2.  **Scene Intersection:** Sample the depth buffer at the current pixel's UV coordinate to reconstruct the 3D position of the scene surface [S3, unofficial].
3.  **Ray Clamping:** If the ray does not hit the scene, clamp the maximum ray distance to a predefined `zfar` value to prevent infinite marching [S3, unofficial].
4.  **Jittering:** Add a pseudo-random delta step $\delta$ to the ray's starting position to mitigate slice artifacts [S3, unofficial].
5.  **Ray Marching Loop:** Iterate from $t=0$ to $t=t_f$ with a fixed step size $\Delta l$ [S3, unofficial].
6.  **In-Scattering Calculation:** At each step position $x$, calculate the in-scattering term $L_i$ [S3, unofficial].
7.  **Transmittance Calculation:** Calculate the transmittance $T$ from the camera to the current step position [S3, unofficial].
8.  **Accumulation:** Add the product of in-scattering, transmittance, and step size to the accumulated radiance [S3, unofficial].
9.  **Early-Out:** If transmittance drops below a threshold, terminate the loop early [S3, unofficial].

**Formulas:**
The final radiance $L$ is approximated using a Riemann sum [S3, unofficial]:
$$
L = \sum_{n=0}^{N} L_i(x(l_n), \omega) e^{-\tau l_n} \Delta l
$$
Where:
*   $L$: Calculated radiance.
*   $N$: Total number of steps.
*   $L_i(x(l_n), \omega)$: In-scattering term at step $n$.
*   $e^{-\tau l_n}$: Transmittance from camera to distance $l_n$.
*   $\tau$: Density (extinction coefficient).
*   $l_n$: Distance traveled along the ray at step $n$.
*   $\Delta l$: Ray-march step size.

The in-scattering term $L_i$ for a point light is defined as [S3, unofficial]:
$$
L_i(x, \omega) = \frac{a \Phi}{4\pi d^2} v(x) e^{-\tau d} P(\omega, \omega')
$$
Where:
*   $a$: Scattering coefficient (probability of photon scattering).
*   $\Phi$: Point light intensity.
*   $d$: Distance from the sample point $x$ to the light source.
*   $v(x)$: Visibility of the sample point from the light source (1 if unshadowed).
*   $e^{-\tau d}$: Transmittance from the light to the sample point.
*   $P(\omega, \omega')$: Phase function describing scattering direction probability.

**Parameters:**
*   **Density ($\tau$):** Constant for uniform fog [S3, unofficial]. Can be procedurally generated or sampled from textures/OpenVDB for non-uniform fog [S3, unofficial].
*   **Visibility ($v(x)$):** Set to 1 in basic implementations; can be sampled from a light's cube shadow map for soft shadows [S3, unofficial].
*   **Phase Function ($P$):** Uses the Schlick phase function approximation of the Henyey-Greenstein (HG) function for real-time performance, replacing the more expensive original HG function [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def ray_marched_volumetric_fog(camera_pos, uv, depth_buffer, light_pos, light_intensity, fog_density, scattering_coeff, step_size, zfar, g):
    # 1. Reconstruct scene intersection [G]
    depth = depth_buffer.sample(uv)
    scene_pos = reconstruct_world_position(uv, depth)
    
    # 2. Calculate ray direction and length
    ray_dir = normalize(scene_pos - camera_pos)
    ray_length = length(scene_pos - camera_pos)
    
    # 3. Clamp ray length
    if ray_length > zfar:
        ray_length = zfar
        
    # 4. Initialize accumulation
    accumulated_radiance = vec3(0.0)
    num_steps = math.ceil(ray_length / step_size)
    
    # 5. Jitter start position to reduce slice artifacts [S3, unofficial]
    jitter = random.uniform(0.0, 1.0) * step_size
    
    # 6. March along the ray
    for n in range(num_steps):
        l = n * step_size + jitter
        current_pos = camera_pos + ray_dir * l
        
        # 7. Calculate distance to light
        d = length(current_pos - light_pos)
        
        # 8. Calculate In-Scattering
        visibility = 1.0  # Basic implementation [S3, unofficial]
        cos_theta = dot(ray_dir, normalize(light_pos - current_pos))
        phase = schlick_henyey_greenstein(cos_theta, g)
        
        in_scattering = (scattering_coeff * light_intensity / (4.0 * math.pi * d * d)) * \
                        visibility * \
                        math.exp(-fog_density * d) * \
                        phase
                        
        # 9. Calculate Transmittance from camera to current step
        transmittance = math.exp(-fog_density * l)
        
        # 10. Accumulate
        accumulated_radiance += in_scattering * transmittance * step_size
        
        # 11. Early-out optimization [S3, unofficial]
        if transmittance < 0.01:
            break
            
    return accumulated_radiance

def reconstruct_world_position(uv, depth):
    # Standard inverse view-projection reconstruction [G]
    # Assumes uv in [-1, 1], depth in [0, 1], and standard OpenGL/DirectX conventions
    pass

def schlick_henyey_greenstein(cos_theta, g):
    # Schlick approximation of Henyey-Greenstein phase function [G]
    g_sq = g * g
    denom = 1.0 + g_sq - 2.0 * g * cos_theta
    return (1.0 - g_sq) / (4.0 * math.pi * (denom ** 1.5))
```

**Edge Cases and Pitfalls:**
*   **Slice Artifacts:** Marching rays with the same step size starting at the same point causes samples to align on the same "slices," creating visible banding [S3, unofficial].
*   **Jittering Trade-off:** Jittering reduces slice artifacts but can introduce noise, especially with larger step sizes [S3, unofficial].
*   **Infinite Marching:** Rays that do not hit the scene must be clamped by `zfar` to prevent infinite loops [S3, unofficial].
*   **Camera Mode Restriction:** Only functions in Perspective and Free camera modes; disabled in Orthographic/Isometric modes [S1].
*   **Ground Plane Bounds:** The ground plane is rendered as infinitely large in all camera modes when enabled, altering scene bounds [S1].

## Subtopics

### [Ray-marched Volumetric Fog with Local Volumes and PCF Shadows](21362-ray-marched-volumetric-fog-with-local-volumes-and-pcf-shadow.md)
Renders HDR height fog and volumetric direct-light scattering by integrating exponential height density and up to 16 transformed box volumes along depth-bounded rays. Distinguishes itself by performing volume shadow lookups using hardware PCF at the actual sample point, eliminating light leakage and removing the cost of repeating raster surface soft-shadow filters at every march step. Operates as a full-resolution, fixed-step, single-scattering pass executed before TAA, bloom, and tone mapping.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of steps $N$ per pixel ($O(N)$). $N$ depends on the distance to the scene and the step size [S3, unofficial].
*   **Memory Complexity:** $O(1)$ per pixel for the accumulation, plus $O(W \times H)$ for the depth buffer [S3, unofficial].
*   **Performance:** Real-time capable on modern GPUs when using fixed step sizes and Schlick phase approximations [S3, unofficial]. Performance degrades with smaller step sizes or complex visibility calculations (shadow mapping) [S3, unofficial].
*   **Strengths:** Simple implementation, allows early-out optimization, dynamic interaction with scene geometry and lighting [S3, unofficial]. Scatters all light types (Sun, Sky, Area) and creates light shafts [S1].
*   **Limitations:** Slice artifacts require jittering or variable step sizes; noise increases with jittering; uniform fog assumption simplifies density but limits complex volumetric structures without additional data structures [S3, unofficial].
*   **When to Use:** Real-time applications requiring dynamic fog, light shafts, and aerial perspective where performance is critical [S3, unofficial]. Enables glass and metal materials to receive reflection and refraction from sun light, and allows background rendering with minimal fog density [S1].
*   **When Not to Use:** Offline rendering requiring high-fidelity multi-scattering or complex non-uniform volumes without precomputed data structures [S3, unofficial].

## Background
The technique is based on the principles of volume rendering and participating media, specifically the work by Toth and Umenhoffer on "Real-time Volumetric Lighting in Participating Media" [S3, unofficial]. It has been implemented in games such as *Horizon Zero Dawn*, *Death Stranding*, and *Returnal* to simulate dynamic fog and light shafts [S3, unofficial]. The forward ray-marching approach was chosen over backward ray-marching to facilitate early-out optimizations [S3, unofficial]. The method is also known as Fog Scattering in CRYENGINE 3, where it was optimized by removing unused branches and supports emissive area lighting [S1].

## Sources
[S1] MagicaVoxel .vox, https://ephtracy.github.io/, fetched 2026-10-26
[S2] DF3 Viewer Oosawa, https://qcganime.web.fc2.com/OOSAWA/MANUAL/FilterE.html, fetched 2026-10-29
[S3] Lon van Ettinger | Ray-marching full screen volumetric fog, https://lonvanettinger.com/portfolio-pages/fog-ray-march-article-1, fetched 2026-10-30 (unofficial)
