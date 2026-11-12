# Single Scattering Volume Rendering with Ray Marching

Also called: Single Scattering Ray Marching, Volumetric Ray Marching, Single-Scattering Volume Rendering with Fixed-Length Ray Marching

## Overview
Single Scattering Volume Rendering with Ray Marching is a real-time technique for rendering participating media (fog, smoke, dust) by simulating light interaction with particles, assuming light scatters only once before reaching the camera [S1][S4, unofficial]. It solves the problem of realistic volumetric lighting and light shafts by integrating the volume rendering equation along camera rays, using shadow maps for occlusion and fixed-length stepping for accumulation [S1][S3, unofficial]. It differs from screen-space god ray techniques by handling multiple light sources, varying density, and physically correct behavior from any viewing angle, and differs from [Basic Volume Rendering and Ray Marching Loop with Absorption & Scattering](6821-basic-volume-rendering-and-ray-marching-loop-with-absorption.md) by explicitly detailing the shadow map sampling and throughput accumulation logic for heterogeneous media [S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Camera position $c$, surface point $p$ (on the opaque geometry behind the volume), surface radiance $L_o(p,v)$, light positions $p_{light_i}$, light intensities/colors, shadow maps, volume density field (e.g., SDF or voxel grid), scattering coefficient $\sigma_s$, absorption coefficient $\sigma_a$, and extinction coefficient $\sigma_t = \sigma_a + \sigma_s$.
*   **Outputs:** Final radiance $L_i(c, -v)$ at the camera.

**Data Structures:**
*   **Shadow Map:** Depth texture storing distance from light to nearest opaque surface.
*   **Volume Density Field:** Can be a Signed Distance Function (SDF) or a 3D texture/voxel grid. For SDF, density is derived from distance to boundary [S4, unofficial].
*   **Ray Parameters:** Origin, direction, length, and step size.

**Formulas:**
1.  **Volume Rendering Equation (Single Scattering Approximation):**
    $$ L_i(c,-v) = T_r(p,c)L_o(p,v) + \int_{0}^{d} T_r(x_t,c) \left[L_e^V(x_t,v) + \sigma_s(x_t)\int_{S^2}f_p(x_t,w,v)L_i(x_t,w) dw \right]dt $$
    Where $c$ is camera position, $p$ is surface point, $d=|c-p|$, $v=(c-p)/d$, $x_t=c-vt$ for $0 \leq t \leq d$, $T_r$ is transmittance, $L_o$ is surface radiance, $L_e^V$ is volume emittance, $\sigma_s$ is scattering coefficient, and $f_p$ is phase function [S2, unofficial].
2.  **Transmittance (Beer-Lambert Law):**
    $$ T_r(p,c) = e^{-\tau}, \quad \tau = \int_{p}^{c} \sigma_e(x) |dx| $$
    Where $\sigma_e = \sigma_a + \sigma_s$ is the extinction coefficient [S2, unofficial].
3.  **In-Scattering (Local Illumination Approximation):**
    Assuming point lights and single scattering, the in-scattering term is approximated as:
    $$ L_{scat}(x_t,v) = \pi \sigma_s(x_t) \sum_{i=1}^{n} f_p(x_t,l_i,v) f_v(x_t,p_{light_i}) L_{light_i}(x_t) $$
    Where $l_i$ is direction to light $i$, $f_v$ is visibility function, and $L_{light_i}$ is radiance from light $i$ [S2, unofficial].
4.  **Visibility Function:**
    $$ f_v(x_t,p_{light_i}) := S(x_i,p_{light_i}) T_r(x_i,p_{light_i}) $$
    Where $S$ is shadow function (occlusion by opaque objects) and $T_r$ is transmittance from light to point [S2, unofficial].
5.  **Henyey-Greenstein Phase Function:**
    $$ f_p(g) = \frac{1}{4\pi} \frac{1-g^2}{(1+g^2-2g\cos\theta)^{3/2}} $$
    Where $g$ is the asymmetry parameter and $\theta$ is the scattering angle [S3, unofficial].

**Algorithm Steps:**
1.  **Initialize:** Given camera $c$, surface $p$, and $L_o(p,v)$. Calculate ray direction $v = (c-p)/|c-p|$ and length $d = |c-p|$ [S2, unofficial].
2.  **Camera Ray Marching:** Iterate $n$ steps along the camera ray from $c$ to $p$.
    *   Calculate sample position $x_t = c - v \cdot t$, where $t$ increases from $0$ to $d$.
    *   Calculate step size $step = d/n$ [S2, unofficial].
    *   Compute transmittance from $x_t$ to $c$: $T_r(x_t,c) = \exp(-\sum \sigma_e \cdot step)$ [S2, unofficial].
    *   For each light $i$:
        *   Calculate direction to light $l_i$.
        *   Compute visibility $f_v(x_t, p_{light_i})$ by marching from $x_t$ to $p_{light_i}$ (see Light Ray Marching below) or sampling shadow map [S2, S3, unofficial].
        *   Compute phase $f_p(x_t, l_i, v)$ using Henyey-Greenstein [S3, unofficial].
        *   Accumulate in-scattering: $L_{scat} += \sigma_s \cdot f_p \cdot f_v \cdot L_{light_i}$ [S3, unofficial].
    *   Accumulate total in-scattering and emittance: $L_a += T_r(x_t,c) \cdot (L_{scat} + L_e^V) \cdot step$ [S2, unofficial].
    *   Update total optical depth $\tau$ for surface transmittance [S2, unofficial].
3.  **Light Ray Marching (Self-Shadowing):** To compute $T_r(x_i, p_{light_i})$ and $S(x_i, p_{light_i})$:
    *   March $k$ steps from $x_t$ towards $p_{light_i}$.
    *   Accumulate absorption: $T_r = \exp(-\sum \sigma_a \cdot step)$ [S4, unofficial].
    *   Check shadow map for occlusion by opaque objects $S$ [S3, unofficial].
    *   Combine: $f_v = S \cdot T_r$ [S2, unofficial].
4.  **Final Computation:**
    *   Compute surface transmittance $T_r(p,c) = \exp(-\tau)$ [S2, unofficial].
    *   Compute surface contribution $L_{surf} = T_r(p,c) \cdot L_o(p,v)$ [S2, unofficial].
    *   Return $L_i = L_{surf} + L_a$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_scattering_volume_rendering(camera_pos, surface_pos, surface_radiance, lights, shadow_map, volume_density_func, sigma_s, sigma_a, g):
    # 1. Initialize Ray Parameters
    ray_dir = normalize(camera_pos - surface_pos)
    ray_length = length(camera_pos - surface_pos)
    n_steps = 30  # Typical value for camera ray [S2, unofficial]
    step_size = ray_length / n_steps
    
    tau = 0.0
    accum_light = vec3(0.0)
    
    # 2. Camera Ray Marching
    for i in range(n_steps):
        t = float(i) * step_size
        sample_pos = camera_pos - ray_dir * t
        
        # Calculate transmittance from sample_pos to camera
        # Note: In a full implementation, tau is accumulated incrementally
        # Here we assume we have the transmittance from sample_pos to camera
        # based on the integral from camera to sample_pos
        tr_to_camera = exp(-tau)
        
        # 3. In-Scattering Calculation
        local_scattering = vec3(0.0)
        for light in lights:
            light_dir = normalize(light.position - sample_pos)
            
            # Visibility: Shadow Map Check + Light Ray Transmittance
            # Shadow Map Check for opaque occlusion
            in_shadow = sample_shadow_map(shadow_map, light, sample_pos)
            
            # Light Ray Marching for volumetric self-shadowing
            light_visibility = get_light_visibility(sample_pos, light_dir, light.position, volume_density_func, sigma_a)
            
            if not in_shadow:
                phase = henyey_greenstein(dot(ray_dir, light_dir), g)
                scattering = sigma_s * phase * light_visibility * light.intensity
                local_scattering += scattering
        
        # Accumulate In-Scattering weighted by transmittance to camera
        accum_light += tr_to_camera * local_scattering * step_size
        
        # Update Optical Depth for Surface Transmittance
        sigma_t = sigma_a + sigma_s
        tau += sigma_t * step_size
        
    # 4. Final Computation
    tr_surface = exp(-tau)
    l_surf = tr_surface * surface_radiance
    
    return l_surf + accum_light

def get_light_visibility(origin, direction, light_pos, volume_density_func, sigma_a):
    # March from origin to light_pos
    light_dist = length(light_pos - origin)
    k_steps = 15  # Typical value for light ray [S2, unofficial]
    step_size = light_dist / k_steps
    
    tau_light = 0.0
    for i in range(k_steps):
        t = float(i) * step_size
        pos = origin + direction * t
        density = volume_density_func(pos)
        tau_light += sigma_a * density * step_size
        
    return exp(-tau_light)
```

**Parameters:**
*   $n$ (camera ray steps): 30 [S2, unofficial].
*   $n$ (light ray steps): 15 [S2, unofficial].
*   $g$ (Henyey-Greenstein asymmetry): Typically 0.8 for forward scattering in fog/smoke [S3, unofficial].
*   $\sigma_a$ (Absorption Coefficient): Low for clear water, high for milk [S4, unofficial].

**Edge Cases and Pitfalls:**
*   **Light Leaking:** Opaque objects rendered before the volume do not account for volume shadowing. Fix by deferring opaque lighting or using volume visibility for opaque objects [S4, unofficial].
*   **Hard Edges:** Volumes with sharp SDF boundaries look unnatural. Soften edges by modulating density with SDF distance [S4, unofficial].
*   **Precision:** Sum $L_{is}$ and $L_e$ into an accumulator $L_a$ before adding to $L_{surf}$ to improve computational precision [S2, unofficial].
*   **Performance:** Complexity is $O(n \cdot m \cdot k)$ where $n$ is camera steps, $m$ is lights, $k$ is light steps [S4, unofficial].

## Subtopics

### [Volumetric Lighting with Phase Functions](2358-volumetric-lighting-with-phase-functions.md)
Computes illumination of participating media by estimating light transport from source to volume sample to camera. Solves for light shafts and silver linings. Differs from [Basic Volume Rendering and Ray Marching Loop with Absorption & Scattering](6821-basic-volume-rendering-and-ray-marching-loop-with-absorption.md) by explicitly integrating the path from volume sample to light source using phase functions, rather than relying on pre-computed ambient terms [2358-volumetric-lighting-with-phase-functions.md].

### [Basic Volume Rendering and Ray Marching Loop with Absorption & Scattering](6821-basic-volume-rendering-and-ray-marching-loop-with-absorption.md)
Renders participating media assuming single scattering, avoiding multi-scattering costs. Differs from [Single Scattering Volume Rendering with Ray Marching](1532-single-scattering-volume-rendering-with-ray-marching.md) by explicitly detailing the ray marching loop structure, distance sampling, and throughput accumulation for homogeneous and heterogeneous media [6821-basic-volume-rendering-and-ray-marching-loop-with-absorption.md].

### Optimization Strategies for Volumetric Lighting
*   **Half-resolution rendering:** Compute god rays at half/quarter resolution for 4x+ performance gain [S3, unofficial].
*   **Bilateral upsampling:** Use depth-aware bilateral filter to avoid halos at depth discontinuities [S3, unofficial].
*   **Temporal reprojection:** Blend with previous frame's result using different sample offsets to multiply sample count [S3, unofficial].
*   **Blue noise dithering:** Offset ray march starting positions with blue noise to convert banding to noise [S3, unofficial].
*   **Early termination:** Stop marching when transmittance drops below threshold (e.g., 0.01) [S3, unofficial].

## Operating characteristics
*   **Complexity:** $O(n \cdot m \cdot k)$, where $n$ is camera ray steps, $m$ is number of lights, and $k$ is light ray steps [S4, unofficial].
*   **Performance:** More expensive than screen-space methods but handles multiple lights, colored fog, varying density, and physically correct behavior from any viewing angle [S3, unofficial].
*   **Strengths:** Physically accurate single scattering, handles heterogeneous media, supports multiple light sources.
*   **Limitations:** Computationally intensive, does not simulate multi-scattering (e.g., distant fog blurring), requires shadow maps or light visibility calculations.
*   **Usage:** Used in Unreal Engine 5 (built-in volumetric fog), Unity HDRP, Red Dead Redemption 2, and The Last of Us Part II [S3, unofficial].

## Background
Originated from GPU-based algorithms for real-time volumetric lighting in participating media [S1]. The method approximates the radiative transport equation by ignoring multiple scattering, making the in-scattering term independent of unknown radiance [S1]. It has been implemented in major engines like Unreal Engine 5 and Unity HDRP, and games like Red Dead Redemption 2 and The Last of Us Part II [S3, unofficial].

## Sources
[S1] Real-time Volumetric Lighting in Participating Media, https://doi.org/10.2312/egs.20091048, fetched 2026-10-26
[S2] 【メモ】リアルタイムなVolume RenderingをGLSLで実装する(1), https://qiita.com/Renard_Renard/items/fd706def17cb8d1e8ec4, fetched 2026-10-27 (unofficial)
[S3] Volumetric Lighting (God Rays) — How It Works - Moonjump, https://moonjump.com/game-dev-mechanics-volumetric-lighting-god-rays-how-it-works/, fetched 2026-10-28 (unofficial)
[S4] Volumetric Rendering Part 2, https://wallisc.github.io/rendering/2020/05/02/Volumetric-Rendering-Part-2.html, fetched 2026-10-29 (unofficial)
