# Monte Carlo Path Tracing for Volume Rendering

Also called: Exposure Render DVR Framework, MCSRender, Monte Carlo Scattering Rendering, Progressive Stochastic Volume Rendering, Stochastic Raycasting DVR

## Overview
Monte Carlo Path Tracing for Volume Rendering is a unified, brute-force, progressive stochastic rendering framework that produces photo-realistic images of volumetric data without relying on pre-computed quantities or auxiliary volumes [S1]. It distinguishes itself from traditional ray marching by using Woodcock tracking to efficiently handle light attenuation and shadows, and by integrating depth-of-field and hybrid surface/volumetric scattering through stochastic sampling rather than slice-based approximations [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Volumetric data (typically 16-bit unsigned short), camera parameters (MVP matrix, lens aperture), lighting configuration (environment, point, area lights), transfer function (opacity $\alpha$, color), and maximum extinction coefficient $\sigma_{max}$ [S1][S2, unofficial].
*   **Output:** A Low Dynamic Range (LDR) image, accumulated progressively over frames [S1].

**Data Structures:**
*   **Accumulator Buffer:** Stores the running Monte Carlo estimate of the High Dynamic Range (HDR) color [S1].
*   **Frame Buffer:** Stores the current frame's HDR estimate [S1].
*   **Camera Parameters:** Stored in constant memory for fast access [S1].
*   **Lights:** Stored in constant memory [S1].

**Numbered Steps:**

1.  **Ray Generation:**
    *   Unproject screen position to get ray origin `vRayFrom` and ray end `vRayTo` using the inverse MVP matrix [S2, unofficial].
    *   Construct a camera ray using a thin lens model: sample a point on the lens and a perturbed point within the pixel area on the film plane using stratified sampling [S1].
    *   Intersect the ray with the volume’s bounding box to obtain parametric range $[t_{bounds}.x, t_{bounds}.y]$ [S1][S2, unofficial].
    *   If $t_{bounds}.x \ge t_{bounds}.y$, the ray misses the volume; sample the environment map and output as color [S2, unofficial].

2.  **Initialization:**
    *   Calculate ray segment `from` and `to` within the cube and `maxDistance` [S2, unofficial].
    *   Initialize seed: `seed = vPosition + rand(vec2(uOffset, uOffset))` [S2, unofficial].
    *   Initialize `from` to the entry point of the volume [S2, unofficial].

3.  **Scattering Point Selection (Woodcock Tracking):**
    *   To find the first scattering point, use Woodcock tracking [S1]:
        1.  Generate a path length $s$ using the maximum extinction coefficient $\sigma_{max}$ [S1].
        2.  Accept the tentative collision point with probability $\sigma_t(p(s)) / \sigma_{max}$, where $\sigma_t$ is the extinction coefficient at point $p(s)$ [S1].
        3.  Repeat until a collision is accepted [S1].
    *   Alternatively, implemented via `sampleDistance(from, to, seed)` [S2, unofficial]:
        1.  Initialize `dist = 0.0`, `invSigmaMax = 1.0 / uSigmaMax`, `invMaxDistance = 1.0 / maxDistance` [S2, unofficial].
        2.  Loop: generate random `seed`, update `dist -= log(1.0 - seed.x) * invSigmaMax` [S2, unofficial].
        3.  If `dist > maxDistance`, break loop [S2, unofficial].
        4.  Calculate `samplingPosition = mix(from, to, dist * invMaxDistance)` [S2, unofficial].
        5.  Get `alphaSample = transferSample.a * uAlphaCorrection` [S2, unofficial].
        6.  If `seed.y < alphaSample * invSigmaMax`, break loop [S2, unofficial].
    *   If the loop breaks due to `dist > maxDistance`, sample the environment map and output [S2, unofficial].

4.  **Lighting and Transmittance Calculation:**
    *   Update `from` to the scattering point: `from = mix(from, to, dist / maxDistance)` [S2, unofficial].
    *   Calculate transmittance from `from` to `to` using `sampleTransmittance(from, to, seed)` [S2, unofficial]:
        1.  Initialize `transmittance = 1.0` [S2, unofficial].
        2.  Loop similarly to `sampleDistance`, updating `transmittance *= 1.0 - alphaSample * invSigmaMax` [S2, unofficial].
    *   Compute direct lighting contribution [S1]:
        1.  Form light ray $R_{light}$ connecting scattering point $p_s$ to a stochastically sampled point on a stochastically chosen light source [S1].
        2.  Apply Woodcock tracking to $R_{light}$ to determine if blocked. If the resulting scattering point $p'_s$ is beyond $p_s$, the ray is unblocked; add light's exitant radiance $L_e$ [S1].
        3.  Sample scattering function yielding scattering ray $R_{scattering}$; intersect with light to compute contribution [S1].
        4.  Combine contributions using Multiple Importance Sampling (MIS) with the power heuristic [S1].
    *   Hybrid Scattering Decision [S1]:
        1.  Draw random number $j$ from uniform distribution.
        2.  Compare $j$ to $P_{brdf}$. If $j < P_{brdf}$, treat as surface scattering (BRDF); otherwise, treat as volumetric scattering (Phase function) [S1].
        3.  $P_{brdf} = \alpha_x \cdot (1 - \exp(-25 \cdot g \cdot \|\nabla_{tn}(x)\|))$, where $\alpha_x$ is opacity, $\|\nabla_{tn}(x)\|$ is normalized gradient magnitude, and $g \in [0,1]$ is gradient factor [S1].
        4.  For surface scattering, use Fresnel blend shading model [S1]. For volumetric, use isotropic phase function (or Schlick's) [S1].

5.  **Color Accumulation:**
    *   Sample diffuse color at `from`: `diffuseColor = sampleVolumeColor(from)` [S2, unofficial].
    *   Sample light color: `lightColor = sampleEnvironmentMap(uScatteringDirection)` [S2, unofficial].
    *   Calculate final color: $oColor = diffuseColor \times lightColor \times transmittance$ [S2, unofficial].
    *   Accumulate frame: `acc = texture(uAccumulator, vPosition)`, `frame = texture(uFrame, vPosition)` [S2, unofficial].
    *   Update accumulator: `oColor = acc + (frame - acc) * uInvFrameNumber` [S2, unofficial].

6.  **Post-Processing:**
    *   Filter HDR MC estimate with separable Gaussian kernel (standard deviation 1.5, window 5x5 pixels) to reduce anti-aliasing [S1].
    *   Tone map HDR estimate to compress dynamic range [S1].
    *   Gamma correct [S1].
    *   An-isotropically filter LDR image to reduce noise [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def monte_carlo_volume_rendering(pixel_pos, camera_params, volume_data, transfer_func, lights, sigma_max, alpha_correction, gradient_factor, frame_number):
    # 1. Ray Generation
    ray_origin, ray_direction = unproject_and_sample_lens(pixel_pos, camera_params)
    t_min, t_max = intersect_bounding_box(ray_origin, ray_direction, volume_data.bbox)
    
    if t_min >= t_max:
        return sample_environment_map(ray_direction)

    max_distance = t_max - t_min
    from_point = ray_origin + t_min * ray_direction
    to_point = ray_origin + t_max * ray_direction
    
    seed = generate_seed(pixel_pos)
    
    # 2. Scattering Point Selection (Woodcock Tracking)
    dist = 0.0
    inv_sigma_max = 1.0 / sigma_max
    inv_max_distance = 1.0 / max_distance
    
    while dist < max_distance:
        r1 = random_float()
        dist -= math.log(1.0 - r1) * inv_sigma_max
        
        if dist >= max_distance:
            break
            
        sampling_pos = mix(from_point, to_point, dist * inv_max_distance)
        alpha_sample = transfer_func.sample_alpha(sampling_pos) * alpha_correction
        
        r2 = random_float()
        if r2 < alpha_sample * inv_sigma_max:
            break
            
    if dist >= max_distance:
        return sample_environment_map(ray_direction)
        
    scattering_point = mix(from_point, to_point, dist * inv_max_distance)
    
    # 3. Transmittance and Lighting
    transmittance = 1.0
    # Simplified transmittance loop for brevity; in practice, similar to distance sampling
    # but accumulating product of (1 - alpha_sample * inv_sigma_max)
    
    # Hybrid Scattering
    r_hybrid = random_float()
    grad_mag = compute_normalized_gradient(sampling_pos)
    p_brdf = transfer_func.sample_alpha(sampling_pos) * (1.0 - math.exp(-25.0 * gradient_factor * grad_mag))
    
    if r_hybrid < p_brdf:
        # Surface scattering
        color = sample_fresnel_blend_brdf(scattering_point, ray_direction, lights)
    else:
        # Volumetric scattering
        scattering_dir = sample_phase_function(ray_direction)
        light_color = sample_environment_map(scattering_dir)
        diffuse_color = transfer_func.sample_color(scattering_point)
        
        # Light attenuation via Woodcock tracking on light ray
        is_unblocked = check_light_shadow(scattering_point, lights)
        if is_unblocked:
            color = diffuse_color * light_color * transmittance
        else:
            color = vec4(0.0)
            
    # 4. Accumulation
    current_acc = get_accumulator_buffer(pixel_pos)
    current_frame = color
    inv_frame_number = 1.0 / frame_number
    new_acc = current_acc + (current_frame - current_acc) * inv_frame_number
    set_accumulator_buffer(pixel_pos, new_acc)
    
    # 5. Post-Processing (Applied after accumulation or per frame depending on pipeline)
    # Gaussian filter, tone mapping, gamma correction, anisotropic filter
    return tone_map_gamma_correct(new_acc)

def check_light_shadow(scattering_point, lights):
    for light in lights:
        # Woodcock tracking on ray from scattering_point to light sample
        # If scattering point yielded by tracking is beyond scattering_point, unblocked
        pass
    return True # Simplified
```

**Parameters:**
*   `sigma_max` (uSigmaMax): Maximum scattering coefficient used for distance sampling [S2, unofficial].
*   `alpha_correction` (uAlphaCorrection): Correction factor for alpha values from transfer function [S2, unofficial].
*   `gradient_factor` (g): Controls surface/volumetric scattering amount, range [0-1] [S1].
*   `uOffset`: Offset for random seed generation [S2, unofficial].
*   `uInvFrameNumber`: Inverse of current frame number for exponential moving average accumulation [S2, unofficial].
*   Gaussian kernel: Standard deviation 1.5, window 5x5 pixels [S1].

**Edge Cases and Pitfalls:**
*   **Noise at Startup:** Objectionable noise occurs at the start of the MC algorithm. A noise reduction filter is applied during initial iterations, increasing blurring initially, with influence reducing based on the error in the running estimate [S1].
*   **Ray Marching Performance:** Standard Ray Marching becomes prohibitively slow for shadow calculations at every sample. Woodcock tracking is used to yield far fewer shadow computations and remains unbiased [S1].
*   **Reset Condition:** If transfer function, lighting, or camera position changes, the Monte Carlo integrated estimate buffer must be cleared and the algorithm restarted [S1].

## Operating characteristics
*   **Time Complexity:** Progressive convergence. Performance is measured in frame estimates per second.
*   **Memory Complexity:** Insensitive to lighting, material, camera, and transfer function configurations. Does not require pre-computed quantities or additional volumes [S1].
*   **Measured Performance (800x600 resolution, 16-bit unsigned short data):**
    *   Mecanix (512x743x512): 35.2 (Front), 34.1 (Left), 36.3 (Top) fps [S1].
    *   Manix (512x460x512): 32.7 (Front), 39.7 (Left), 42.4 (Top) fps [S1].
    *   Engine (256x256x128): 65.5 (Front), 53.7 (Left), 65.7 (Top) fps [S1].
    *   Bonsai (256x256x257): 57.7 (Front), 66.6 (Left), 68.1 (Top) fps [S1].
    *   Artifix (512x347x512): 47.3 (Front), 40.3 (Left), 50.8 (Top) fps [S1].
*   **Strengths:** Photo-realism, unified handling of isosurfaces and volumes, support for complex lighting (directional, point, environment), depth-of-field via stochastic sampling, unbiased [S1].
*   **Limitations:** Noise at low frame counts requires filtering; progressive nature means no single-frame instant result without accumulation [S1].

## Background
*   **Origin:** Developed by Csébfalvi et al. as part of the Exposure Render framework [S1].
*   **History:** Csébfalvi et al. were the first to apply Monte Carlo integration to volume rendering to solve depth-of-field effects and achieve photo-realism [S1]. The method was demonstrated on datasets including Mecanix, Manix, Engine, Bonsai, Artifix, and a bat CT scan [S1].
*   **Usage:** Implemented in the Exposure Render framework [S1].

## Sources
[S1] Exposure Render: An Interactive Photo-Realistic Volume Rendering Framework, https://doi.org/10.1371/journal.pone.0038586, fetched 2026-10-26
[S2] UL-FRI-LGM/vpt-conductor, https://github.com/UL-FRI-LGM/vpt-conductor, fetched 2026-10-26 (unofficial)
