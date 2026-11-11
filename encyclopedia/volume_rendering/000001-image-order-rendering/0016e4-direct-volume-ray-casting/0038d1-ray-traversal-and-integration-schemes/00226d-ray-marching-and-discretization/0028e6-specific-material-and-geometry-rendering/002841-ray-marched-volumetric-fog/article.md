# Ray-marched Volumetric Fog

Also called: Forward ray-marching fog, Full-screen volumetric fog

## Overview
Ray-marched volumetric fog simulates light absorption, scattering, soft shadows, and light shafts (god rays) by treating fog as a 3D volume of participating media traced via ray-marching from the camera to the scene [S1, unofficial]. It differs from backward ray-marching (scene to camera) by enabling early-out optimizations when transmittance drops below a threshold, and it operates as a full-screen post-process or screen-space technique that reconstructs 3D positions from depth buffers [S1, unofficial].

## How it works
**Inputs:**
*   Camera position and view/projection matrices.
*   Depth buffer (z-buffer) containing scene depth values.
*   Light parameters: position, intensity ($\Phi$), and optionally shadow maps.
*   Fog parameters: density ($\tau$), scattering coefficient ($a$), step size ($\Delta l$), and phase function parameters.
*   Ray termination distance ($z_{far}$).

**Outputs:**
*   Final radiance $L$ per pixel, composited over the background scene color.

**Data Structures:**
*   No complex 3D data structures (like VDB or grids) are required for uniform fog; density is a scalar constant [S1, unofficial].
*   For non-uniform fog, density $\tau(x)$ is sampled from a texture, procedural function, or volume file (e.g., OpenVDB) [S1, unofficial].

**Numbered Steps:**

1.  **Ray Generation:** For each pixel with UV coordinates $(u, v)$, reconstruct the 3D world position $P_{hit}$ of the scene intersection by sampling the depth buffer at $(u, v)$ and unprojecting the depth value using the inverse view-projection matrix [S1, unofficial].
2.  **Ray Definition:** Define the ray origin $O$ as the camera position and direction $D$ as the vector from $O$ to $P_{hit}$. Normalize $D$.
3.  **Interval Clamping:** Calculate the total ray length $L_{total} = \|P_{hit} - O\|$. Clamp this length to a maximum $z_{far}$ to prevent infinite marching if the ray misses the scene geometry: $L_{max} = \min(L_{total}, z_{far})$ [S1, unofficial].
4.  **Jittering (Optional but Recommended):** To mitigate slice artifacts, add a pseudo-random offset $\delta \in [0, \Delta l)$ to the starting distance $t_{start} = \delta$ [S1, unofficial].
5.  **Integration Loop:** Iterate from $t = t_{start}$ to $t = L_{max}$ with step size $\Delta l$. For each step $n$:
    *   **Position:** Calculate current position $x_n = O + D \cdot t_n$.
    *   **Density:** Retrieve local density $\tau(x_n)$. For uniform fog, $\tau(x_n) = \tau_{const}$ [S1, unofficial].
    *   **Transmittance:** Update accumulated transmittance $T_n = e^{-\sum_{i=0}^{n} \tau(x_i) \Delta l}$. Alternatively, update iteratively: $T_{n} = T_{n-1} \cdot e^{-\tau(x_n) \Delta l}$ [S1, unofficial].
    *   **In-Scattering:** Calculate in-scattered radiance $L_i(x_n, \omega)$ [S1, unofficial]:
        $$ L_i(x_n, \omega) = \frac{a \Phi}{4\pi d^2} v(x_n) e^{-\tau(x_n) d} P(\omega, \omega_i) $$
        Where:
        *   $d$ is the distance from $x_n$ to the light source.
        *   $v(x_n)$ is the visibility of $x_n$ from the light (1.0 if no shadow mapping is used) [S1, unofficial].
        *   $P(\omega, \omega_i)$ is the phase function (e.g., Schlick approximation of Henyey-Greenstein) [S1, unofficial].
    *   **Accumulation:** Add to total radiance: $L \leftarrow L + L_i(x_n, \omega) \cdot T_n \cdot \Delta l$ [S1, unofficial].
6.  **Termination:** If $T_n$ drops below a threshold (e.g., 0.01), break the loop early [S1, unofficial].
7.  **Composition:** Composite the accumulated fog radiance $L$ with the background color $C_{bg}$ sampled from the scene buffer: $C_{final} = C_{bg} \cdot T_{final} + L$ [S1, unofficial].

**Formulas:**

The total radiance $L$ is approximated using a Riemann sum [S1, unofficial]:
$$ L = \sum_{n=0}^{N} L_i(x(l_n), \omega) e^{-\tau l_n} \Delta l $$
Where:
*   $L_i(x(l_n), \omega)$ is the in-scattering term at step $n$.
*   $e^{-\tau l_n}$ is the transmittance for the distance $l_n$ traveled from the camera.
*   $\Delta l$ is the ray-march step size [S1, unofficial].

The in-scattering term $L_i$ for a point light is [S1, unofficial]:
$$ L_i(x, \omega) = \frac{a \Phi}{4\pi d^2} v(x) e^{-\tau d} P(\omega, \omega_i) $$
Where:
*   $\tau$ is the density.
*   $a$ is the scattering coefficient.
*   $\Phi$ is the point light intensity.
*   $d$ is the distance from the sample point to the light.
*   $v(x)$ is the visibility (shadow factor).
*   $P(\omega, \omega_i)$ is the phase function [S1, unofficial].

**Parameters:**
*   **Density ($\tau$):** Constant for uniform fog; typically a small positive value [S1, unofficial].
*   **Step Size ($\Delta l$):** Controls resolution and performance. Larger steps increase speed but cause slice artifacts and noise [S1, unofficial].
*   **Phase Function:** Schlick approximation of Henyey-Greenstein is used for real-time performance [S1, unofficial].
*   **Visibility ($v(x)$):** Set to 1.0 in basic implementations; can be sampled from a light's cube shadow map for soft shadows [S1, unofficial].

**Pitfalls:**
*   **Slice Artifacts:** Occur when rays march with the same step size starting at the same point, causing samples to align on discrete "slices" [S1, unofficial]. Mitigated by jittering the starting position or every sample [S1, unofficial].
*   **Noise:** Jittering can introduce noise, especially with larger step sizes [S1, unofficial].
*   **Infinite Marching:** Rays missing the scene must be clamped to $z_{far}$ [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_marched_volumetric_fog(u, v, camera_pos, view_proj_inv, depth_buffer, light_pos, light_intensity, density, step_size, z_far, scattering_coeff, phase_param):
    # 1. Reconstruct 3D position from depth
    depth = depth_buffer.sample(u, v)
    if depth == 1.0: # No hit
        return background_color(u, v)
    
    # Unproject to get world position
    clip_pos = vec4(u * 2 - 1, v * 2 - 1, depth * 2 - 1, 1)
    world_pos = inverse(view_proj_inv) * clip_pos
    world_pos /= world_pos.w
    
    # 2. Define Ray
    ray_dir = normalize(world_pos - camera_pos)
    ray_len = length(world_pos - camera_pos)
    
    # 3. Clamp Ray
    max_len = min(ray_len, z_far)
    
    # 4. Jitter Start
    jitter = random_float(0, step_size)
    t = jitter
    
    # 5. Initialize Accumulators
    total_radiance = vec3(0.0)
    transmittance = 1.0
    
    # 6. Integration Loop
    while t < max_len and transmittance > 0.01:
        # Position
        pos = camera_pos + ray_dir * t
        
        # Density (Uniform)
        tau = density
        
        # Transmittance Update
        # Note: The formula uses e^(-tau * l_n) where l_n is total distance from camera.
        # We can compute this directly or iteratively. Direct is safer for clarity.
        current_transmittance = exp(-tau * t)
        
        # In-Scattering Calculation
        dist_to_light = length(pos - light_pos)
        light_attenuation = light_intensity / (4 * pi * dist_to_light * dist_to_light)
        
        # Visibility (1.0 if no shadow map)
        visibility = 1.0
        
        # Light Transmittance (from light to sample point)
        light_transmittance = exp(-tau * dist_to_light)
        
        # Phase Function (Schlick approx of HG)
        # Simplified placeholder for P(w, wi)
        phase = schlick_henyey_greenstein(phase_param)
        
        in_scattering = scattering_coeff * light_attenuation * visibility * light_transmittance * phase
        
        # Accumulate
        total_radiance += in_scattering * current_transmittance * step_size
        
        # Advance
        t += step_size
        
    # 7. Composition
    bg_color = background_color(u, v)
    final_color = bg_color * exp(-tau * max_len) + total_radiance
    return final_color
```

## Operating characteristics
*   **Time Complexity:** $O(N \cdot P)$, where $N$ is the number of pixels and $P$ is the number of steps per ray. Performance is heavily dependent on step size and early-out efficiency.
*   **Memory Complexity:** $O(1)$ per pixel for uniform fog, as no 3D volume data is stored in GPU memory during the march (only depth buffer and light info) [S1, unofficial].
*   **Strengths:** Simple to implement; works well for uniform fog; early-out optimization reduces cost for distant or opaque regions; produces soft shadows and light shafts [S1, unofficial].
*   **Limitations:** Slice artifacts require jittering; noise increases with larger step sizes; performance degrades with high density or many steps; does not handle complex volume interactions (like multiple scattering) efficiently [S1, unofficial].
*   **When to Use:** Real-time applications requiring atmospheric effects (fog, smoke) with dynamic lighting; when memory bandwidth is a constraint for large 3D textures [S1, unofficial].
*   **When Not to Use:** High-fidelity offline rendering requiring accurate multiple scattering; scenes with highly complex, non-uniform volumes where ray-marching is too slow compared to voxel-based methods [S1, unofficial].

## Background
This technique is used in games like *Horizon Zero Dawn*, *Death Stranding*, and *Returnal* to simulate cinematic fog that interacts with light and geometry [S1, unofficial]. It is based on the principles of participating media rendering, specifically adapting the forward ray-marching approach described in "Real-time Volumetric Lighting in Participating Media" by Toth and Umenhofer [S1, unofficial]. The forward approach was chosen over backward ray-marching for its early-out optimization potential [S1, unofficial].

## Sources
[S1] Lon van Ettinger | Ray-marching full screen volumetric fog, https://lonvanettinger.com/portfolio-pages/fog-ray-march-article-1, fetched 2026-10-30 (unofficial)
