# Ray-marched Volumetric Fog

Also called: Forward ray-marching fog, Full-screen volumetric fog

## Overview
Ray-marched volumetric fog simulates light absorption, scattering, soft shadows, and light shafts (god rays) by treating fog as a 3D volume of participating media traced via ray-marching from the camera to the scene [S1, unofficial]. It differs from backward ray-marching (scene to camera) by enabling early-out optimizations when transmittance becomes negligible, and it differs from voxel-based methods by operating directly in screen space using depth reconstruction rather than explicit volume data structures [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Camera view ray direction, camera position, depth buffer (for scene intersection), light position, light intensity ($\Phi$), fog density ($\tau$), scattering coefficient ($a$), and step size ($\Delta l$).
*   **Outputs:** Final pixel color (radiance $L$).

**Data Structures**
*   **Depth Buffer:** 2D texture or array storing depth values $z$ for each pixel UV coordinate.
*   **Ray State:** Current position $x$, accumulated transmittance $T$, accumulated radiance $L_{acc}$.

**Numbered Steps**
1.  **Ray Initialization:** For each pixel with UV coordinate $u$, shoot a ray from the camera origin $C$ through the view direction $\omega$.
2.  **Scene Intersection:** Reconstruct the 3D position of the scene intersection point $x_{end}$ by sampling the depth buffer at $u$ and unprojecting the depth value [S1, unofficial].
3.  **Ray Clamping:** Clamp the total ray length to a maximum distance $z_{far}$ to prevent infinite marching if the ray misses the scene geometry [S1, unofficial].
4.  **Jittering (Optional):** Add a pseudo-random delta step $\delta$ to the ray's starting position to fix slice artifacts [S1, unofficial].
5.  **Marching Loop:** Iterate from $n = 0$ to $N$ along the ray from camera to $x_{end}$:
    a. Calculate current sample position $x(l_n)$.
    b. Calculate distance $d$ from $x(l_n)$ to the light source.
    c. Calculate visibility $v(x)$ from $x(l_n)$ to the light source.
    d. Calculate in-scattering $L_i$.
    e. Calculate transmittance $e^{-\tau l_n}$ from camera to $x(l_n)$.
    f. Accumulate radiance: $L_{acc} += L_i \cdot e^{-\tau l_n} \cdot \Delta l$.
6.  **Final Compositing:** Combine accumulated fog radiance with the background scene color (not detailed in source, implied by "full-screen shader").

**Formulas**
The total radiance $L$ is approximated using a Riemann sum [S1, unofficial]:

$$ L = \sum_{n=0}^{N} L_i(x(l_n), \omega) e^{-\tau l_n} \Delta l $$

Where:
*   $L$: Calculated radiance.
*   $L_i(x(l_n), \omega)$: In-scattering term at step $n$.
*   $e^{-\tau l_n}$: Transmittance for distance $l_n$ travelled from camera.
*   $\Delta l$: Ray-march step size.
*   $\tau$: Fog density.

The in-scattering term $L_i$ for a point light is [S1, unofficial]:

$$ L_i(x, \omega) = \frac{a \Phi}{4\pi d^2} v(x) e^{-\tau d} P(\omega, \omega') $$

Where:
*   $a$: Scattering coefficient (probability of photon scattering).
*   $\Phi$: Point light intensity.
*   $d$: Distance between sample point $x$ and light source.
*   $v(x)$: Visibility of sample point from light source.
*   $e^{-\tau d}$: Transmittance from light to sample point.
*   $P(\omega, \omega')$: Phase function describing scattering direction probability.

**Parameters**
*   **Density ($\tau$):** Constant for uniform fog [S1, unofficial].
*   **Visibility ($v(x)$):** Set to 1 in the reference implementation [S1, unofficial]. Can be sampled from a light's cube shadow map if available.
*   **Phase Function ($P$):** Uses Schlick phase function approximation of Henyey-Greenstein (HG) for real-time performance [S1, unofficial].
*   **Step Size ($\Delta l$):** Determines resolution of the march. Larger steps increase noise but reduce cost.

**Pitfalls**
*   **Slice Artifacts:** Occur when rays march with the same step size starting at the same point, causing samples to align on the same "slices" [S1, unofficial].
*   **Noise:** Jittering reduces slice artifacts but can make the result look noisier, especially with larger step sizes [S1, unofficial].
*   **Banding:** Visible in the distance, partially mitigated by jittering every sample on a ray [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_marched_volumetric_fog(uv, camera_pos, view_direction, depth_buffer, light_pos, light_intensity, density, scattering_coeff, step_size, z_far):
    # 1. Reconstruct scene intersection
    depth = depth_buffer.sample(uv)
    scene_pos = unproject(uv, depth) # [G] Standard unprojection using camera matrices
    
    # 2. Calculate ray direction and length
    ray_dir = normalize(scene_pos - camera_pos)
    ray_len = length(scene_pos - camera_pos)
    
    # 3. Clamp ray length
    if ray_len > z_far:
        ray_len = z_far
        
    # 4. Jitter start position to fix slice artifacts
    jitter = random_float() * step_size # [G] Pseudo-random value in [0, step_size)
    start_offset = jitter * ray_dir
    
    # 5. Initialize accumulation
    accumulated_radiance = vec3(0.0)
    current_pos = camera_pos + start_offset
    
    # 6. Marching loop
    num_steps = int(ray_len / step_size)
    for n in range(num_steps):
        # Current sample position
        sample_pos = camera_pos + (n * step_size + jitter) * ray_dir
        
        # Distance from sample to light
        d = length(light_pos - sample_pos)
        
        # Visibility (set to 1 if no shadow mapping)
        visibility = 1.0 # [S1, unofficial]
        
        # Transmittance from camera to sample
        l_n = length(sample_pos - camera_pos)
        transmittance_camera = exp(-density * l_n)
        
        # Transmittance from light to sample
        transmittance_light = exp(-density * d)
        
        # Phase function (Schlick approximation of Henyey-Greenstein)
        # [G] Simplified Schlick phase function P(g, theta)
        # g is asymmetry parameter, theta is angle between view and light direction
        # Assuming g=0 for isotropic or pre-calculated for HG approx
        phase = schlick_phase_function(view_direction, normalize(light_pos - sample_pos))
        
        # In-scattering term
        # L_i = (a * Phi / (4 * pi * d^2)) * v(x) * e^(-tau * d) * P
        in_scattering = (scattering_coeff * light_intensity / (4.0 * 3.14159 * d * d)) * visibility * transmittance_light * phase
        
        # Accumulate: L_i * transmittance_from_camera * step_size
        accumulated_radiance += in_scattering * transmittance_camera * step_size
        
    return accumulated_radiance
```

## Subtopics

### [Procedural Volumetric Cloud Rendering](3480-procedural-volumetric-cloud-rendering.md)
Renders volumetric clouds using procedural noise functions. Choose when clouds are the primary volumetric element and require complex internal structure.

### [Mr-Voxel Volume Rendering](3874-mr-voxel-volume-rendering.md)
Renders volumes using voxel grids and marching cubes/rays. Choose when explicit volumetric data (e.g., medical scans, smoke simulations) is available and high accuracy is needed.

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of pixels and $P$ is the number of march steps per ray.
*   **Memory Complexity:** $O(1)$ per pixel (excluding depth buffer and light shadow maps).
*   **Performance:** Real-time capable on modern GPUs if step count is limited. Performance degrades with higher resolution, larger step sizes, or complex light interactions (shadow maps).
*   **Strengths:** Simple implementation, works with existing scene geometry, allows dynamic lighting interactions (soft shadows, god rays).
*   **Limitations:** Slice artifacts without jittering, noise with large steps, no occlusion by other volumetric objects (only scene geometry), uniform fog assumption in basic implementation.
*   **When to use:** Real-time games requiring atmospheric fog with dynamic lighting.
*   **When not to use:** Offline rendering requiring high accuracy, scenes with complex non-uniform volumetric data (use voxel methods), or when performance is extremely constrained and simple 2D fog is sufficient.

## Background
The technique is based on the paper "Real-time Volumetric Lighting in Participating Media" by Toth and Umenhofer [S1, unofficial]. It has been implemented in games such as Horizon Zero Dawn, Death Stranding, and Returnal [S1, unofficial]. The implementation described is a forward ray-marching approach, deviating from the backward ray-marching in the original paper to allow for early-out optimizations [S1, unofficial].

## Sources
[S1] Lon van Ettinger | Ray-marching full screen volumetric fog, https://lonvanettinger.com/portfolio-pages/fog-ray-march-article-1, fetched 2026-10-30 (unofficial)
