# Monte Carlo Path Tracing for Volume Rendering

Also called: Exposure Render DVR Framework, MCSRender, Monte Carlo Scattering Rendering, Progressive Stochastic Volume Rendering, Stochastic Raycasting DVR

## Overview
Monte Carlo Path Tracing for Volume Rendering is a unified, brute-force, progressive stochastic rendering framework that computes photo-realistic direct volume rendering (DVR) without relying on pre-computed quantities or auxiliary data structures [S1]. It solves the problem of integrating complex lighting, shadows, and hybrid surface/volumetric scattering in participating media by treating the volume as a continuous medium where light transport is estimated via random sampling [S1][S2, unofficial]. Unlike slice-based or layer-based methods that treat isosurfaces distinctly, this approach renders the volume uniformly, supporting directional, point, and environment lights while maintaining a memory footprint insensitive to lighting or material complexity [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   Volumetric dataset encoded in 16-bit unsigned short format [S1].
*   Camera parameters (position, orientation, lens aperture, film plane dimensions) stored in constant memory [S1].
*   Transfer function mapping scalar values to opacity $\alpha$ and color [S1][S2, unofficial].
*   Lighting configuration (environment map, point lights, area lights) [S1].
*   Maximum extinction coefficient $\sigma_{max}$ (uniform float `uSigmaMax`) [S2, unofficial].
*   Alpha correction factor `uAlphaCorrection` [S2, unofficial].
*   Gradient factor `g` for hybrid scattering [S1].

**Outputs:**
*   A Low Dynamic Range (LDR) image, progressively refined over frames [S1].
*   An accumulator buffer storing the High Dynamic Range (HDR) Monte Carlo estimate [S1][S2, unofficial].

### Data Structures
*   **Accumulator Buffer:** A texture buffer storing the cumulative HDR color estimate per pixel, initialized to zero [S2, unofficial].
*   **Frame Buffer:** A texture buffer storing the current frame's HDR Monte Carlo estimate [S2, unofficial].
*   **Camera Parameters:** Stored in CUDA constant memory for fast access during ray generation [S1].
*   **Light Data:** Stored in CUDA constant memory [S1].

### Algorithm Steps

1.  **Initialization and Reset:**
    *   If the transfer function, lighting, or camera position changes, clear the accumulator buffer to zero and reset the frame counter [S1][S2, unofficial].
    *   Initialize the accumulator buffer $A$ and frame buffer $F$ to zero [S2, unofficial].

2.  **Ray Generation:**
    *   Unproject screen coordinates to obtain ray origin $v_{RayFrom}$ and ray end $v_{RayTo}$ using the inverse Model-View-Projection (MVP) matrix [S2, unofficial].
    *   Construct rays using a thin lens camera model: sample a point on the lens and a perturbed point on the film plane using stratified sampling to simulate depth of field [S1].
    *   Intersect the ray with the volume's bounding box to determine the parametric range $[t_{start}, t_{end}]$. Discard rays that do not intersect [S1].
    *   Calculate the ray segment within the volume: $from = v_{RayFrom} + t_{start} \cdot direction$, $to = v_{RayFrom} + t_{end} \cdot direction$. Compute $maxDistance = \|to - from\|$ [S2, unofficial].

3.  **Stochastic Ray Casting (Single Scattering Estimation):**
    *   Initialize random seed: $seed = v_{Position} + rand(vec2(uOffset, uOffset))$ [S2, unofficial].
    *   **Distance Sampling (Woodcock Tracking):**
        *   Initialize $dist = 0.0$, $invSigmaMax = 1.0 / \sigma_{max}$, $invMaxDistance = 1.0 / maxDistance$ [S2, unofficial].
        *   Loop:
            *   Generate random value $r_1$.
            *   Update distance: $dist \leftarrow dist - \ln(1.0 - r_1) \cdot invSigmaMax$ [S2, unofficial].
            *   If $dist > maxDistance$, break loop.
            *   Calculate sampling position: $samplingPosition = mix(from, to, dist \cdot invMaxDistance)$ [S2, unofficial].
            *   Sample volume color at $samplingPosition$ to get $alphaSample$ from the transfer function. Apply correction: $alphaSample \leftarrow alphaSample \cdot uAlphaCorrection$ [S2, unofficial].
            *   Generate random value $r_2$.
            *   If $r_2 < alphaSample \cdot invSigmaMax$, break loop. The loop terminates at the sampled position [S2, unofficial].
    *   **Transmittance Calculation:**
        *   Initialize $transmittance = 1.0$ [S2, unofficial].
        *   Perform a similar loop from $from$ to the accepted scattering point $ps$ (determined by $dist$) [S2, unofficial].
        *   Update transmittance: $transmittance \leftarrow transmittance \cdot (1.0 - alphaSample \cdot invSigmaMax)$ at each step [S2, unofficial].
    *   **Lighting and Hybrid Scattering:**
        *   Sample diffuse color at $ps$: $diffuseColor = sampleVolumeColor(ps)$ [S2, unofficial].
        *   Sample light color from environment map in scattering direction: $lightColor = sampleEnvironmentMap(uScatteringDirection)$ [S2, unofficial].
        *   **Hybrid Scattering Decision:**
            *   Calculate BRDF probability $P_{brdf}$:
                $$ P_{brdf} = \alpha_x \cdot (1 - e^{-25 \cdot g \cdot \|\nabla_{tn}(x)\|}) $$
                where $\alpha_x$ is opacity at $x$, $\|\nabla_{tn}(x)\|$ is the normalized gradient magnitude, and $g \in [0, 1]$ is the gradient factor [S1].
            *   Draw random number $j$ from uniform distribution.
            *   If $j < P_{brdf}$, treat as surface scattering (BRDF); otherwise, treat as volumetric scattering (Phase function) [S1].
        *   **Light Attenuation (Shadowing):**
            *   Use Woodcock tracking to determine if the light ray from $ps$ to the light source is blocked [S1].
            *   If the scattering point $ps'$ yielded by tracking is beyond $ps$, the ray is unblocked; add light's exitant radiance $L_e$. If $ps'$ is before $ps$, the ray is blocked [S1].
        *   **Multiple Importance Sampling (MIS):**
            *   Combine contributions from light ray $R_{light}$ and scattering ray $R_{scattering}$ using the power heuristic [S1].
    *   **Final Color Calculation:**
        *   Compute frame color: $oColor = diffuseColor \times lightColor \times transmittance$ [S2, unofficial].

4.  **Accumulation and Filtering:**
    *   Accumulate the frame estimate into the accumulator buffer using a cumulative moving average:
        $$ acc_{new} = acc_{old} + (frame - acc_{old}) \cdot uInvFrameNumber $$
        where $uInvFrameNumber = 1.0 / currentFrame$ [S2, unofficial].
    *   Filter the HDR MC estimate with a separable Gaussian kernel (standard deviation 1.5, window 5x5 pixels) to reduce anti-aliasing [S1].
    *   Apply anisotropic filtering to the LDR image to reduce noise; the influence of this filtering reduces based on the error in the running estimate [S1].
    *   Tone map the HDR estimate to compress dynamic range [S1].
    *   Apply gamma correction to produce the final LDR image [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def intersect_bounding_box(origin, direction, bbox_min, bbox_max):
    """[G] Simple axis-aligned bounding box intersection."""
    t_min = (bbox_min[0] - origin[0]) / direction[0]
    t_max = (bbox_max[0] - origin[0]) / direction[0]
    if t_min > t_max: t_min, t_max = t_max, t_min
    ty_min = (bbox_min[1] - origin[1]) / direction[1]
    ty_max = (bbox_max[1] - origin[1]) / direction[1]
    if ty_min > ty_max: ty_min, ty_max = ty_max, ty_min
    if t_min > ty_max or ty_min > t_max: return None, None
    if ty_min > t_min: t_min = ty_min
    if ty_max < t_max: t_max = ty_max
    tz_min = (bbox_min[2] - origin[2]) / direction[2]
    tz_max = (bbox_max[2] - origin[2]) / direction[2]
    if tz_min > tz_max: tz_min, tz_max = tz_max, tz_min
    if t_min > tz_max or tz_min > t_max: return None, None
    if tz_min > t_min: t_min = tz_min
    return t_min, t_max

def monte_carlo_path_tracing_volume_rendering(width, height, volume_data, transfer_function, 
                                            environment_map, lights, camera, sigma_max, 
                                            alpha_correction, gradient_factor, offset, bbox_min, bbox_max):
    # Initialize buffers as 3D arrays [height][width][3]
    accumulator = [[[0.0, 0.0, 0.0] for _ in range(width)] for _ in range(height)]
    frame_counter = 0
    
    while True: # [G] Loop continues until rendering_active flag is false
        frame_counter += 1
        inv_frame_number = 1.0 / frame_counter
        
        for y in range(height):
            for x in range(width):
                # 1. Ray Generation
                ray_origin, ray_direction = camera.get_ray(x, y)
                
                # Intersect with volume bounding box
                t_start, t_end = intersect_bounding_box(ray_origin, ray_direction, bbox_min, bbox_max)
                if t_start is None or t_start >= t_end:
                    continue # Ray misses volume
                
                ray_from = (ray_origin[0] + ray_direction[0]*t_start, 
                            ray_origin[1] + ray_direction[1]*t_start, 
                            ray_origin[2] + ray_direction[2]*t_start)
                ray_to = (ray_origin[0] + ray_direction[0]*t_end, 
                          ray_origin[1] + ray_direction[1]*t_end, 
                          ray_origin[2] + ray_direction[2]*t_end)
                max_distance = math.sqrt(sum((ray_to[i]-ray_from[i])**2 for i in range(3)))
                
                # 2. Stochastic Ray Casting (Woodcock Tracking)
                seed = (x + random.uniform(0, offset), y + random.uniform(0, offset))
                dist = 0.0
                inv_sigma_max = 1.0 / sigma_max
                inv_max_distance = 1.0 / max_distance
                
                scattering_point = None
                
                while dist < max_distance:
                    r1 = random.uniform(0, 1)
                    dist -= math.log(1.0 - r1) * inv_sigma_max
                    
                    if dist >= max_distance:
                        break
                    
                    # Sample position along ray
                    t = dist * inv_max_distance
                    pos = (ray_from[0] + (ray_to[0]-ray_from[0])*t,
                           ray_from[1] + (ray_to[1]-ray_from[1])*t,
                           ray_from[2] + (ray_to[2]-ray_from[2])*t)
                    
                    # Sample volume properties
                    scalar_val = volume_data.sample(pos)
                    color, alpha = transfer_function(scalar_val)
                    alpha_corrected = alpha * alpha_correction
                    
                    r2 = random.uniform(0, 1)
                    if r2 < alpha_corrected * inv_sigma_max:
                        scattering_point = pos
                        break
                
                if scattering_point is None:
                    # Ray escaped volume without scattering
                    frame_color = environment_map(ray_direction)
                else:
                    # 3. Transmittance Calculation
                    transmittance = 1.0
                    current_dist = 0.0
                    
                    # Re-trace to scattering point for transmittance
                    while current_dist < dist:
                        r1 = random.uniform(0, 1)
                        step = -math.log(1.0 - r1) * inv_sigma_max
                        current_dist += step
                        
                        if current_dist > dist:
                            break
                        
                        t = current_dist * inv_max_distance
                        pos = (ray_from[0] + (ray_to[0]-ray_from[0])*t,
                               ray_from[1] + (ray_to[1]-ray_from[1])*t,
                               ray_from[2] + (ray_to[2]-ray_from[2])*t)
                        scalar_val = volume_data.sample(pos)
                        _, alpha = transfer_function(scalar_val)
                        alpha_corrected = alpha * alpha_correction
                        
                        transmittance *= (1.0 - alpha_corrected * inv_sigma_max)
                    
                    # 4. Lighting and Hybrid Scattering
                    diffuse_color = volume_data.sample_color(scattering_point)
                    
                    # Determine scattering direction (simplified for single scattering)
                    scattering_dir = random.sample_phase_function(ray_direction)
                    light_color = environment_map(scattering_dir)
                    
                    # Hybrid Scattering
                    grad_mag = volume_data.compute_normalized_gradient(scattering_point)
                    alpha_x = transfer_function(volume_data.sample(scattering_point)).alpha
                    p_brdf = alpha_x * (1.0 - math.exp(-25.0 * gradient_factor * grad_mag))
                    
                    if random.uniform(0, 1) < p_brdf:
                        # Surface scattering (BRDF)
                        brdf_color = compute_fresnel_blend(scattering_point, ray_direction, scattering_dir)
                        final_color = brdf_color * transmittance
                    else:
                        # Volumetric scattering
                        # Check shadowing using Woodcock tracking
                        is_unblocked = check_shadow(scattering_point, lights, sigma_max)
                        if is_unblocked:
                            final_color = diffuse_color * light_color * transmittance
                        else:
                            final_color = (0.0, 0.0, 0.0)
                    
                    frame_color = final_color
                
                # 5. Accumulation
                acc_color = accumulator[y][x]
                new_acc = (acc_color[0] + (frame_color[0] - acc_color[0]) * inv_frame_number,
                           acc_color[1] + (frame_color[1] - acc_color[1]) * inv_frame_number,
                           acc_color[2] + (frame_color[2] - acc_color[2]) * inv_frame_number)
                accumulator[y][x] = new_acc
        
        # 6. Post-Processing
        # [G] Filter HDR estimate with Gaussian kernel (sigma=1.5, 5x5 window)
        # [G] Tone map and gamma correct
        # [G] Anisotropic filtering for noise reduction
        # [G] Display final image
```

### Parameters
*   `uSigmaMax`: Maximum scattering coefficient for Woodcock tracking [S2, unofficial].
*   `uAlphaCorrection`: Correction factor for alpha values from the transfer function [S2, unofficial].
*   `uOffset`: Offset for random seed generation [S2, unofficial].
*   `uInvFrameNumber`: Inverse of the current frame number for exponential moving average accumulation, computed as $1.0 / frame$ [S2, unofficial].
*   `g`: Gradient factor controlling surface/volumetric scattering amount, range $[0, 1]$ [S1].
*   Gaussian kernel standard deviation: 1.5 [S1].
*   Gaussian kernel window size: 5x5 pixels [S1].

### Edge Cases and Pitfalls
*   **Noise at Startup:** The Monte Carlo algorithm produces objectionable noise in initial iterations. A noise reduction filter is applied during these iterations, which increases blurring. The influence of this filter reduces based on the error in the running estimate [S1].
*   **Reset Condition:** Any change in transfer function, lighting, or camera position requires clearing the accumulator buffer and restarting the MC algorithm from the beginning [S1].

## Operating characteristics

### Time and Memory Complexity
*   **Time Complexity:** The method is iterative and progressive. Performance is measured in frame estimates per second. It does not have a fixed convergence time but improves with more frames [S1].
*   **Memory Complexity:** The memory footprint is insensitive to lighting, material, camera, and transfer function configurations [S1].

### Measured Performance
Performance measured in frame estimates per second at 800x600 resolution with environment light and two additional area lights [S1]:

| Dataset | Size | Front | Left | Top |
| :--- | :--- | :--- | :--- | :--- |
| Mecanix | 512x743x512 | 35.2 | 34.1 | 36.3 |
| Manix | 512x460x512 | 32.7 | 39.7 | 42.4 |
| Engine | 256x256x128 | 65.5 | 53.7 | 65.7 |
| Bonsai | 256x256x257 | 57.7 | 66.6 | 68.1 |
| Artifix | 512x347x512 | 47.3 | 40.3 | 50.8 |

### Strengths and Limitations
*   **Strengths:**
    *   Unified rendering of volume and surface scattering without treating isosurfaces differently [S1].
    *   Supports complex lighting (directional, point, environment) and shadows [S1].
    *   Unbiased estimation using Woodcock tracking [S1].
    *   Memory footprint is independent of lighting complexity [S1].
*   **Limitations:**
    *   Initial iterations are noisy, requiring filtering that causes temporary blurring [S1].
    *   Progressive nature means no single frame is a final solution; requires accumulation over time [S1].

### When to Use
*   Use when photo-realistic lighting, shadows, and hybrid surface/volumetric scattering are required [S1].
*   Use when the volume data does not have explicit isosurfaces or when a unified treatment of the volume is desired [S1].
*   Use when interactive refinement is acceptable, and the user can wait for convergence [S1].

### When Not to Use
*   Use when real-time, single-frame rendering is required without progressive refinement [S1].
*   Use when the volume is extremely sparse and deterministic ray marching with early termination is significantly faster [G].
*   Use when hardware does not support GPU-based Monte Carlo integration efficiently [G].

## Background

### Origin and History
The method was introduced by Csébfalvi et al. in 2012 as part of the Exposure Render framework [S1]. It was one of the first demonstrations of interactive, brute-force, progressive stochastic rendering for photo-realism in Direct Volume Rendering (DVR) [S1]. The work built upon earlier applications of Monte Carlo integration to volume rendering by Csébfalvi et al. [27] and incorporated techniques like Woodcock tracking [31] and Multiple Importance Sampling (MIS) [33].

### Development
The framework was developed to address limitations in previous DVR methods:
*   Unlike Salama et al. [6], it does not render layers using isosurfaces as a basis for scattering [S1].
*   Unlike Schott et al. [29], it integrates depth-of-field using stochastic sampling rather than a slice-based approach with incremental filtering [S1].
*   Unlike Schlegel et al. [28] and Ropinski et al. [2], it supports multiple light types (directional, point, environment) rather than being limited to a single point light source [S1].

### Usage
The method is used in the Exposure Render framework and has been demonstrated on various datasets including Mecanix, Manix, Engine, Bonsai, Artifix, and a bat CT scan [S1].

## Sources

[S1] Exposure Render: An Interactive Photo-Realistic Volume Rendering Framework, https://doi.org/10.1371/journal.pone.0038586, fetched 2026-10-26
[S2] UL-FRI-LGM/vpt-conductor, https://github.com/UL-FRI-LGM/vpt-conductor, fetched 2026-10-26 (unofficial)
