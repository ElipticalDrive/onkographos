# Ray-marched Volumetric Fog

Also called: Fog Filter, Forward ray-marching fog, Full-screen volumetric fog, Volumetric Fog (CRYENGINE 3), Voxel-based Volumetric Fog

## Overview
Ray-marched volumetric fog simulates aerial perspective and light shafts by treating fog as a 3D volume of participating media traced via forward ray-marching from the camera to the scene [S2, unofficial]. It solves the problem of rendering dynamic, light-interacting fog that absorbs and scatters light, distinguishing itself from backward ray-marching by allowing early-out optimizations when transmittance becomes negligible [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position, view direction per pixel, depth buffer (for scene intersection), light position and intensity, fog density ($\tau$), scattering coefficient ($a$), and phase function parameters.
*   **Outputs:** Final radiance $L$ per pixel.

**Data Structures:**
*   **Ray:** Defined by origin (camera position) and direction (view vector).
*   **Depth Buffer:** 2D texture mapping UV coordinates to depth values $z$.
*   **Light Source:** Point light defined by position and intensity $\Phi$.

**Numbered Steps:**
1.  **Ray Initialization:** For each pixel, construct a ray originating at the camera position.
2.  **Scene Intersection:** Sample the depth buffer at the current pixel's UV coordinate to reconstruct the 3D position of the scene surface. Calculate the distance $t_f$ from the camera to this intersection point [S2, unofficial].
3.  **Ray Clamping:** If the ray does not hit the scene (e.g., background), clamp the maximum ray distance to a predefined `zfar` value to prevent infinite marching [S2, unofficial].
4.  **Jittering (Optional but Recommended):** To mitigate slice artifacts, add a pseudo-random delta step $\delta$ to the ray's starting position or sample offsets [S2, unofficial].
5.  **Ray Marching Loop:** Iterate from $t=0$ to $t=t_f$ with a fixed step size $\Delta l$.
6.  **In-Scattering Calculation:** At each step position $x$, calculate the in-scattering term $L_i$ [S2, unofficial].
7.  **Transmittance Calculation:** Calculate the transmittance $T$ from the camera to the current step position [S2, unofficial].
8.  **Accumulation:** Add the product of in-scattering, transmittance, and step size to the accumulated radiance [S2, unofficial].
9.  **Early-Out:** If transmittance drops below a threshold, terminate the loop early [S2, unofficial].

**Formulas:**
The final radiance $L$ is approximated using a Riemann sum [S2, unofficial]:
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

The in-scattering term $L_i$ for a point light is defined as [S2, unofficial]:
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
*   **Density ($\tau$):** Constant for uniform fog [S2, unofficial]. Can be procedurally generated or sampled from textures/OpenVDB for non-uniform fog [S2, unofficial].
*   **Visibility ($v(x)$):** Set to 1 in basic implementations; can be sampled from a light's cube shadow map for soft shadows [S2, unofficial].
*   **Phase Function ($P$):** Uses the Schlick phase function approximation of the Henyey-Greenstein (HG) function for real-time performance, replacing the more expensive original HG function [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_marched_volumetric_fog(camera_pos, view_dir, depth_buffer, light_pos, light_intensity, fog_density, scattering_coeff, step_size, zfar):
    # 1. Reconstruct scene intersection
    uv = get_pixel_uv()
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
    current_transmittance = 1.0
    
    # 5. Jitter start position to reduce slice artifacts [G]
    jitter = random_float() * step_size
    
    # 6. March along the ray
    for t in range(0, int(ray_length), int(step_size)):
        current_pos = camera_pos + ray_dir * (t + jitter)
        
        # 7. Calculate distance to light
        dist_to_light = length(current_pos - light_pos)
        
        # 8. Calculate In-Scattering
        # Visibility v(x) is 1 in basic implementation [S2, unofficial]
        visibility = 1.0 
        
        # Phase function P (Schlick approximation of HG) [S2, unofficial]
        # Note: Exact Schlick formula parameters not provided in source, 
        # assuming standard implementation of Schlick-HG approximation.
        phase = schlick_henyey_greenstein(ray_dir, normalize(light_pos - current_pos))
        
        in_scattering = (scattering_coeff * light_intensity / (4.0 * 3.14159 * dist_to_light * dist_to_light)) * \
                        visibility * \
                        exp(-fog_density * dist_to_light) * \
                        phase
                        
        # 9. Calculate Transmittance from camera to current step
        transmittance = exp(-fog_density * t)
        
        # 10. Accumulate
        accumulated_radiance += in_scattering * transmittance * step_size
        
        # 11. Early-out optimization [S2, unofficial]
        if transmittance < 0.01:
            break
            
    return accumulated_radiance
```

**Edge Cases and Pitfalls:**
*   **Slice Artifacts:** Marching rays with the same step size starting at the same point causes samples to align on the same "slices," creating visible banding [S2, unofficial].
*   **Jittering Trade-off:** Jittering reduces slice artifacts but can introduce noise, especially with larger step sizes [S2, unofficial].
*   **Infinite Marching:** Rays that do not hit the scene must be clamped by `zfar` to prevent infinite loops [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of steps $N$ per pixel ($O(N)$). $N$ depends on the distance to the scene and the step size.
*   **Memory Complexity:** $O(1)$ per pixel for the accumulation, plus $O(W \times H)$ for the depth buffer.
*   **Performance:** Real-time capable on modern GPUs when using fixed step sizes and Schlick phase approximations [S2, unofficial]. Performance degrades with smaller step sizes or complex visibility calculations (shadow mapping) [S2, unofficial].
*   **Strengths:** Simple implementation, allows early-out optimization, dynamic interaction with scene geometry and lighting [S2, unofficial].
*   **Limitations:** Slice artifacts require jittering or variable step sizes; noise increases with jittering; uniform fog assumption simplifies density but limits complex volumetric structures without additional data structures [S2, unofficial].
*   **When to Use:** Real-time applications requiring dynamic fog, light shafts, and aerial perspective where performance is critical.
*   **When Not to Use:** Offline rendering requiring high-fidelity multi-scattering or complex non-uniform volumes without precomputed data structures.

## Background
The technique is based on the principles of volume rendering and participating media, specifically the work by Toth and Umenhofer on "Real-time Volumetric Lighting in Participating Media" [S2, unofficial]. It has been implemented in games such as *Horizon Zero Dawn*, *Death Stranding*, and *Returnal* to simulate dynamic fog and light shafts [S2, unofficial]. The forward ray-marching approach was chosen over backward ray-marching to facilitate early-out optimizations [S2, unofficial].

## Sources
[S1] DF3 Viewer Oosawa, https://qcganime.web.fc2.com/OOSAWA/MANUAL/FilterE.html, fetched 2026-10-29
[S2] Lon van Ettinger | Ray-marching full screen volumetric fog, https://lonvanettinger.com/portfolio-pages/fog-ray-march-article-1, fetched 2026-10-30 (unofficial)
