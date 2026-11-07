# Single Scattering Volume Rendering with Ray Marching

Also called: Single Scattering Ray Marching, Volumetric Ray Marching, Single-Scattering Volume Rendering with Fixed-Length Ray Marching

## Overview
Single Scattering Volume Rendering with Ray Marching is a real-time technique for computing volumetric light shafts (god rays) and fog in participating media by assuming light scatters only once before reaching the camera [S1][S4, unofficial]. It solves the problem of simulating light interaction with particles (dust, fog, smoke) in 3D space, handling multiple light sources, varying density, and self-shadowing, which screen-space methods cannot [S3, unofficial]. It sets itself apart from screen-space radial blur techniques by providing physically correct behavior from any viewing angle and supporting colored fog and multiple lights, at the cost of higher computational complexity [S3, unofficial]. It builds on [Path Tracing and Sampling](2526-path-tracing-and-sampling.md) by restricting the path length to a single scattering event and using fixed-length ray marching for integration.

## How it works
The method approximates the radiative transport equation by ignoring multiple scattering, meaning the in-scattering term $L_i$ no longer depends on the unknown radiance $L$ [S1]. The rendering equation for a camera ray from position $c$ to surface point $p$ with direction $v = (c-p)/d$ (where $d=|c-p|$) is:

$$ L_i(c,-v) = T_r(p,c)L_o(p,v) + \int_{0}^{d} T_r(x_t,c) \left[L_e^V(x_t,v) + \sigma_s(x_t)\int_{S^2}f_p(x_t,w,v)L_i(x_t,w) dw \right]dt $$

where $x_t = c - vt$ for $0 \leq t \leq d$, $T_r$ is transmittance, $L_o$ is surface radiance, $L_e^V$ is volume emittance, $\sigma_s$ is the scattering coefficient, and $f_p$ is the phase function [S2, unofficial].

### Inputs and Outputs
*   **Inputs:** Camera position $c$, surface point $p$, surface radiance $L_o(p,v)$, light positions and intensities, volume density field (e.g., SDF or voxel grid), extinction coefficient $\sigma_e = \sigma_a + \sigma_s$, scattering coefficient $\sigma_s$, phase function parameters (e.g., $g$), and shadow maps for opaque occlusion [S2, S4, unofficial].
*   **Outputs:** Final radiance $L_i(c,-v)$ for the pixel [S2, unofficial].

### Data Structures
*   **Ray Parameters:** Origin $c$, direction $v$, length $d$ [S2, unofficial].
*   **Accumulators:** `accumLight` (in-scattering), `transmittance` (camera ray attenuation), `tau` (optical depth) [S2, S3, unofficial].
*   **Light Visibility:** Computed via separate ray march towards light source [S4, unofficial].

### Numbered Steps
1.  **Initialize Ray Parameters:** Given camera $c$ and surface $p$, calculate distance $d = |c-p|$, direction $v = (c-p)/d$, and step size $step = d/n$, where $n$ is the number of march steps (typically 30) [S2, unofficial].
2.  **Initialize Accumulators:** Set `accumLight` = 0, `transmittance` = 1.0, and `tau` = 0.0 [S2, S3, unofficial].
3.  **Camera Ray Marching Loop:** Iterate $i$ from 0 to $n-1$:
    a. Calculate sample position $x_t = c - v \cdot (i \cdot step)$ [S2, unofficial].
    b. Update optical depth: `tau` += $\sigma_e(x_t) \cdot step$ [S2, unofficial].
    c. Calculate current transmittance from sample to camera: $T_r(x_t,c) = \exp(-\text{tau})$ [S2, unofficial].
    d. **In-Scattering Calculation:**
        i. For each light $i$ (typically 3):
            1. Calculate light direction $l_i$ and distance.
            2. Compute visibility $f_v(x_t, p_{light_i}) = S(x_t, p_{light_i}) \cdot T_r(x_t, p_{light_i})$, where $S$ is shadow function (opaque occlusion) and $T_r$ is transmittance from light to point [S2, unofficial].
            3. Compute transmittance from light to point by marching $k$ steps (typically 15) towards the light, accumulating absorption using Beer-Lambert Law [S2, S4, unofficial].
            4. Compute phase function value $f_p(x_t, l_i, v)$ (e.g., Henyey-Greenstein) [S3, S5, unofficial].
            5. Calculate scattering contribution: $L_{scat} = \pi \sigma_s(x_t) \sum f_p \cdot f_v \cdot L_{light_i}$ [S2, unofficial].
        ii. Add to accumulator: `accumLight` += $T_r(x_t,c) \cdot L_{scat} \cdot step$ [S2, S3, unofficial].
    e. **Update Transmittance:** Multiply `transmittance` by $\exp(-\sigma_e(x_t) \cdot step)$ [S3, unofficial].
    f. **Optimization:** If `transmittance` < threshold (e.g., 0.01), break loop [S3, S4, unofficial].
4.  **Compute Final Radiance:**
    a. Calculate surface transmittance $T_r(p,c) = \exp(-\text{tau})$ [S2, unofficial].
    b. Calculate surface contribution $L_{surf} = T_r(p,c) \cdot L_o(p,v)$ [S2, unofficial].
    c. Combine terms: $L_a = L_{is} + L_e$ (summed for precision) [S2, unofficial].
    d. Return $L_{surf} + L_a$ [S2, unofficial].

### Formulas
*   **Beer-Lambert Law (Absorption):** $A = e^{-c \cdot d}$, where $A$ is transmitted light fraction, $c$ is absorption coefficient, $d$ is distance [S4, unofficial].
*   **Transmittance:** $T_r(p,c) = e^{-\tau}$, where $\tau = \int_{p}^{c} \sigma_e(x) |dx|$ and $\sigma_e = \sigma_a + \sigma_s$ [S2, unofficial].
*   **Henyey-Greenstein Phase Function:** $P(\theta) = \frac{1 - g^2}{4\pi (1 + g^2 - 2g\cos\theta)^{1.5}}$, where $g$ is asymmetry parameter and $\theta$ is angle between view and light ray [S5, unofficial].
*   **Visibility Function:** $f_v(x_t, p_{light_i}) = S(x_t, p_{light_i}) T_r(x_t, p_{light_i})$ [S2, unofficial].

### Parameters
*   **Ray march steps ($n$):** 30 for camera ray [S2, unofficial].
*   **Shadow ray steps:** 15 for light visibility [S2, unofficial].
*   **Asymmetry parameter ($g$):** ~0.8 for forward scattering, ~-0.2 for backward scattering [S5, unofficial].
*   **Absorption Coefficient:** Material-dependent; low for water, high for milk [S4, unofficial].
*   **Number of lights:** 3 in example implementations [S2, unofficial].

### Edge Cases and Pitfalls
*   **Light Leaking:** Opaque objects rendered before volume do not account for volume shadowing. Fix by calling light visibility function for opaque object lighting [S4, unofficial].
*   **Hard Edges:** Volume edges appear too sharp. Fix by softening density based on SDF distance to volume boundary [S4, unofficial].
*   **Precision:** Sum $L_{is}$ and $L_e$ into accumulator $L_a$ before adding to $L_{surf}$ to improve computational precision [S2, unofficial].
*   **Self-Shadowing:** Without light ray marching, light reaches all parts of volume equally. Must march towards lights to compute self-shadowing [S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def single_scattering_volume_rendering(camera_pos, surface_pos, surface_radiance, lights, volume_density_func, sigma_s, sigma_a, g, shadow_map):
    d = length(camera_pos - surface_pos)
    v = normalize(camera_pos - surface_pos)
    n = 30  # Camera ray steps
    step = d / n
    
    tau = 0.0
    accum_light = vec3(0.0)
    
    for i in range(n):
        t = i * step
        x_t = camera_pos - v * t
        
        # Update optical depth
        sigma_e = sigma_s + sigma_a
        tau += sigma_e * step
        
        # Current transmittance from sample to camera
        T_r_cam = exp(-tau)
        
        # In-scattering calculation
        L_scat = vec3(0.0)
        for light in lights:
            light_dir = normalize(light.position - x_t)
            light_dist = length(light.position - x_t)
            
            # Compute visibility (shadow + light transmittance)
            # March towards light to compute light transmittance
            k = 15  # Light ray steps
            light_step = light_dist / k
            light_tau = 0.0
            for j in range(k):
                t_light = j * light_step
                x_light = x_t + light_dir * t_light
                light_tau += sigma_e(x_light) * light_step
            
            T_r_light = exp(-light_tau)
            
            # Shadow from opaque objects
            S = sample_shadow_map(shadow_map, light, x_light)
            
            visibility = S * T_r_light
            
            # Phase function
            theta = dot(v, light_dir)
            phase = heney_greenstein_phase_function(theta, g)
            
            L_scat += phase * visibility * light.intensity * sigma_s(x_t)
        
        accum_light += T_r_cam * L_scat * step
        
        # Optimization: Early termination
        if T_r_cam < 0.01:
            break
            
    # Final transmittance from surface to camera
    T_r_surf = exp(-tau)
    L_surf = T_r_surf * surface_radiance
    
    # Combine terms
    L_a = accum_light  # Assuming no emission for simplicity
    return L_surf + L_a

def heney_greenstein_phase_function(theta, g):
    cos_theta = cos(theta)
    numerator = 1.0 - g * g
    denominator = 4.0 * 3.14159 * pow(1.0 + g * g - 2.0 * g * cos_theta, 1.5)
    return numerator / denominator
```

## Subtopics

### [Basic Volume Rendering and Ray Marching Loop with Absorption & Scattering](6821-basic-volume-rendering-and-ray-marching-loop-with-absorption.md)
This variant focuses on the core loop of accumulating absorption and scattering along the camera ray, often using Signed Distance Functions (SDFs) to determine volume presence and density. It sets itself apart by explicitly handling density variation via SDFs and softening edges based on distance to volume boundary, whereas the main method may assume uniform density or use voxel grids. Choose this when rendering volumetric fog or clouds with soft, diffuse edges. Link: [Basic Volume Rendering and Ray Marching Loop with Absorption & Scattering](6821-basic-volume-rendering-and-ray-marching-loop-with-absorption.md).

### Volumetric Lighting with Phase Functions
This variant emphasizes the use of phase functions (like Henyey-Greenstein) to model directional scattering, particularly for sunlight through clouds or fog. It sets itself apart by integrating a short ray march towards the light source specifically to compute solar attenuation and applying the phase function for edge highlights (silver linings). Choose this for sky and cloud rendering where forward/backward scattering lobes are visually significant. Link: [Volumetric Lighting with Phase Functions](6821-basic-volume-rendering-and-ray-marching-loop-with-absorption.md) (Note: No separate article exists, so linked to parent/variant context).

### Optimization Strategies for Volumetric Lighting
This variant covers performance techniques such as half-resolution rendering, bilateral upsampling, temporal reprojection, and blue noise dithering. It sets itself apart by focusing on reducing computational cost (O(n*m*k)) through spatial, temporal, and sampling optimizations rather than physical accuracy. Choose this when targeting real-time frame rates on limited hardware. Link: [Optimization Strategies for Volumetric Lighting](6821-basic-volume-rendering-and-ray-marching-loop-with-absorption.md) (Note: No separate article exists, so linked to parent/variant context).

## Operating characteristics
*   **Time Complexity:** O(n * m * k), where n is camera ray steps, m is number of lights, and k is light ray steps [S4, unofficial].
*   **Memory Complexity:** Depends on volume representation (voxel grid, SDF, texture) and shadow maps.
*   **Measured Performance:** More expensive than screen-space methods but handles multiple lights and varying density [S3, unofficial]. Used in Unreal Engine 5, Unity HDRP, Red Dead Redemption 2, and The Last of Us Part II [S3, unofficial].
*   **Strengths:** Physically correct behavior from any viewing angle, supports multiple light sources, colored fog, varying density, and self-shadowing [S3, unofficial].
*   **Limitations:** High computational cost, requires careful optimization (temporal reprojection, early termination) for real-time performance [S3, S4, unofficial].
*   **When to use:** Real-time games requiring high-quality volumetric lighting with multiple lights and dynamic shadows.
*   **When not to use:** Mobile or low-end hardware where screen-space god rays are sufficient, or when multi-scattering effects (distant fog blurring) are critical [S4, unofficial].

## Background
The method originates from real-time volumetric lighting research, with early implementations in games like Crysis (2007) using screen-space techniques [S3, unofficial]. The ray marching approach for single scattering was formalized in GPU-based algorithms for participating media, such as the work by Krivánek et al. (2009) [S1]. It has since been adopted in major engines like Unreal Engine 5 (using froxel grids) and Unity HDRP [S3, unofficial]. The technique is widely used in AAA games like Red Dead Redemption 2 and The Last of Us Part II for realistic fog and light shafts [S3, unofficial].

## Sources
[S1] Real-time Volumetric Lighting in Participating Media, https://doi.org/10.2312/egs.20091048, fetched 2026-10-26
[S2] 【メモ】リアルタイムなVolume RenderingをGLSLで実装する(1), https://qiita.com/Renard_Renard/items/fd706def17cb8d1e8ec4, fetched 2026-10-27 (unofficial)
[S3] Volumetric Lighting (God Rays) — How It Works - Moonjump, https://moonjump.com/game-dev-mechanics-volumetric-lighting-god-rays-how-it-works/, fetched 2026-10-28 (unofficial)
[S4] Volumetric Rendering Part 2, https://wallisc.github.io/rendering/2020/05/02/Volumetric-Rendering-Part-2.html, fetched 2026-10-29 (unofficial)
[S5] Volumetric Clouds with 3D Perlin-Worley Noise in OpenGL, https://salivity.github.io/opengl/article/volumetric-clouds-with-3-d-perlin-worley-noise-in-open-gl, fetched 2026-10-29 (unofficial)
