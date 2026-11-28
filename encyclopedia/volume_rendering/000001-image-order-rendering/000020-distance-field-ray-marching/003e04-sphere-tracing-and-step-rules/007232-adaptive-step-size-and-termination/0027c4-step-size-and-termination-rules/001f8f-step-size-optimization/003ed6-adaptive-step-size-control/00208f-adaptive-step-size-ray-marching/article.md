# Adaptive Step Size Ray Marching

Also called: Adaptive Step Size Volume Rendering, CSRayMarch, DDVR, DDVR Raymarching Shader, Ray Marching with Adaptive Step Size and Temporal Accumulation, Volume Rendering via Ray Marching

## Overview
Adaptive Step Size Ray Marching is a volume rendering technique that optimizes computational cost by varying the sampling interval along a ray based on local density changes and transmittance [S1][S2, S3, S4, unofficial]. Unlike fixed-step linear sampling, which forces a trade-off between quality and efficiency, this method uses fine steps in high-density regions and coarse steps in empty space or at distance [S1][S4, unofficial]. It distinguishes itself from siblings like [Dynamic Sampling Step Adjustment](2442-dynamic-sampling-step-adjustment.md) by explicitly coupling step size to density gradients and transmittance thresholds, and from [Variance-based Step Size Adaptation](13161-variance-based-step-size-adaptation.md) by relying on deterministic density thresholds rather than statistical variance [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin $\mathbf{o}$, normalized direction $\mathbf{d}$, 3D volume texture(s), transfer function/color map, camera parameters, and configuration parameters (step size, thresholds) [S3, S4, unofficial].
*   **Outputs:** Final pixel color (RGB) and alpha (A) [S3, unofficial].

**Data Structures:**
*   **Volume Texture:** 3D texture (`sampler3D`) storing density/scalar values. For 4D volumes, two textures are used: `volumeTexture` and `volumeTexture4DNextFrame` [S3, unofficial].
*   **Noise Image:** 2D image buffer (`image2D`, format `rg32f`) storing per-pixel noise data. Component `.r` stores the current step size (or 0 for default); component `.g` stores the ray offset for jittering [S3, unofficial].
*   **Position Depth Buffer:** 2D image buffer (`image2D`, format `rgba32f`) storing the world position of the first hit (`.xyz`) and depth $t$ (`.w`) [S3, unofficial].
*   **Motion Vector Image:** 2D image buffer (`image2D`, format `rgba8`) storing motion vectors in UV space (`.xy`) [S3, unofficial].
*   **Output Image:** 2D image buffer (`image2D`, format `rgba8`) storing the final rendered color [S3, unofficial].

**Parameters:**
*   `stepSize`: Base step size for ray marching [S3, unofficial].
*   `ZERO_THRESHOLD`: Threshold for density and transmittance to skip empty space or terminate ray [S2, unofficial].
*   `EARLY_STOP_THRESHOLD`: Transmittance value below which the ray march terminates early (e.g., 0.05) [S4, unofficial].
*   `STEPSIZE_FINE`: Maximum size of a fine step [S4, unofficial].
*   `COARSE_STEPSIZE_MULTIPLIER`: Multiplier to determine coarse step size from fine step size [S4, unofficial].
*   `MAX_NUM_MISSED_STEPS`: Threshold for switching from fine to coarse steps after missing cloud density [S4, unofficial].
*   `scalarMinMax`: Vec2 containing min/max scalar values for density normalization [S3, unofficial].
*   `accumulation`: Boolean flag enabling temporal accumulation [S3, unofficial].
*   `bVolume4D`: Boolean flag indicating time-varying volume [S3, unofficial].
*   `pingpong4D`: Boolean flag for swapping 4D volume textures [S3, unofficial].
*   `interpolationFactor`: Float value to mix between two 4D volume frames [S3, unofficial].

**Algorithm Steps:**

1.  **Ray Initialization:**
    *   Construct ray from camera position and UV coordinates [S3, unofficial].
    *   Perform ray-box intersection against the volume bounding box to find entry ($t_{entry}$) and exit ($t_{exit}$) distances [S1][S3, unofficial]. If no intersection, output black color and return [S3, unofficial].
    *   Initialize transmittance $T = 1.0$, accumulated color $C_{acc} = \mathbf{0}$, and light energy $L = 0.0$ [S3, S4, unofficial].
    *   Load noise offset from `noiseImage` for the current pixel. If `.r` is 0, use default `stepSize`; otherwise, use `.r` as `m_stepSize` [S3, unofficial].
    *   Initialize step offset `rayOffset` from `noiseImage.g` [S3, unofficial].

2.  **Step Size Calculation (Cloud-Specific Variant):**
    *   If using cloud rendering with fine/coarse steps [S4, unofficial]:
        *   Calculate `totalDst` = $t_{exit} - t_{entry}$ [S4, unofficial].
        *   Set `curFineStepSize` = $\min(\text{STEPSIZE\_FINE}, \text{totalDst} / \text{MIN\_NUM\_FINE\_STEPS})$ [S4, unofficial].
        *   Set `curCoarseStepSize` = `curFineStepSize` $\times$ `COARSE_STEPSIZE_MULTIPLIER` [S4, unofficial].
        *   Set `curThreshold` = `MAX_NUM_MISSED_STEPS` [S4, unofficial].
        *   Set initial `dt` = `curCoarseStepSize` [S4, unofficial].

3.  **Ray Marching Loop:**
    *   Iterate $t$ from $t_{entry} + \text{rayOffset}$ to $t_{exit}$ with step `m_stepSize` (or `dt` for cloud variant) [S3, S4, unofficial].
    *   **Position Calculation:** Calculate current position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$ [S3, unofficial].
    *   **Volume Sampling:**
        *   Convert $\mathbf{p}$ to texture coordinates `uvw` [S2, S3, unofficial].
        *   If `bVolume4D` is true, sample density from `volumeTexture` and `volumeTexture4DNextFrame`, mixing them using `interpolationFactor` and `pingpong4D` [S3, unofficial].
        *   If `bVolume4D` is false, sample density directly from `volumeTexture` [S3, unofficial].
    *   **Density Threshold Check:** If density $> 0.01$ (or $> 0$ for clouds), proceed to color mapping and compositing [S3, S4, unofficial].
        *   **Cloud Variant:** Decrement `curThreshold`. If `curThreshold` $\le 0$, switch to `curCoarseStepSize` [S4, unofficial].
    *   **Color Mapping & Compositing (if density > threshold):**
        *   Calculate `colorMappingCoord` = clamp((density - `scalarMinMax.x`) / (`scalarMinMax.y` - `scalarMinMax.x`), 0, 1) [S3, unofficial].
        *   Sample color map to get `s` (vec4: RGB color, A extinction) [S3, unofficial].
        *   Calculate alpha: $\alpha = 1.0 - \exp(-\text{extinction} \times \text{m\_stepSize})$ [S3, unofficial].
        *   Calculate contribution: $C_{contrib} = T \times \text{s.rgb} \times \alpha$ [S3, unofficial].
        *   Update accumulated color: $C_{acc} += C_{contrib}$ [S3, unofficial].
        *   Update transmittance: $T *= (1.0 - \alpha)$ [S3, unofficial].
        *   **Lighting (Optional):** If density is above threshold, calculate light contribution (e.g., single scattering toward sun) [S2, unofficial]. For clouds, update light energy: `lightEnergy *= phaseVal; cloudColor = lightEnergy * sunColor` [S4, unofficial].
    *   **First Hit Storage:** If `!firstHit` and $\alpha > 0$, set `firstHit = true` and store $\mathbf{p}$ and $t$ in `positionDepthBufferImage` [S3, unofficial].
    *   **Early Termination:** If $1.0 - T > 0.95$ (or $T < \text{EARLY\_STOP\_THRESHOLD}$), break loop [S3, S4, unofficial].
    *   **Step Size Adaptation (General Variant):**
        *   Calculate density difference `dDensity` = current density - previous density [S2, unofficial].
        *   Compute adaptive step size: `newStep` = `GetStep(dDensity, T, density, stepScale)` [S2, unofficial].
        *   Update step: `step` = (`step` + `newStep`) $\times 0.5$ [S2, unofficial].
        *   Update `prevDensity` = current density [S2, unofficial].
    *   **Advance Position:** $\mathbf{p} += \mathbf{d} \times \text{m\_stepSize}$ (or `dt`) [S3, S4, unofficial].

4.  **Post-Processing:**
    *   Apply gamma correction: $C_{acc}.rgb = \text{pow}(C_{acc}.rgb, 1.0/2.2)$ [S3, unofficial].
    *   Calculate final alpha: $\alpha_{final} = 1.0 - T$ [S3, unofficial].
    *   Apply illumination scale: $C_{final} = C_{acc} \times \text{illuminationScale}$ [S3, unofficial].
    *   **Temporal Accumulation:** If `accumulation` is true and camera has not moved, blend $C_{final}$ with previous frame color using $\alpha_{final}$ as weight [S3, unofficial].
    *   **Motion Vectors:** If enabled, calculate motion vector by reprojecting `positionDepthBufferImage` to previous frame coordinates [S3, unofficial].
    *   **Update Noise:** Update `noiseImage` with new step size and offset for next frame [S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_ray_march(ray_origin, ray_direction, t_entry, t_exit, volume_texture, color_map, config):
    # Initialization
    t = t_entry + config.ray_offset
    transmittance = 1.0
    accumulated_color = [0.0, 0.0, 0.0]
    first_hit = False
    prev_density = 0.0

    # Determine step size from noise image
    m_step_size = config.step_size if config.noise_r == 0 else config.noise_r

    # Cloud-specific variables (if applicable)
    if config.use_cloud_steps:
        total_dst = t_exit - t_entry
        cur_fine_step = min(config.STEPSIZE_FINE, total_dst / config.MIN_NUM_FINE_STEPS)
        cur_coarse_step = cur_fine_step * config.COARSE_STEPSIZE_MULTIPLIER
        cur_threshold = config.MAX_NUM_MISSED_STEPS
        dt = cur_coarse_step
    else:
        dt = m_step_size

    while t < t_exit:
        # Position calculation
        pos = [ray_origin[i] + ray_direction[i] * t for i in range(3)]

        # Volume sampling (explicit coordinate conversion and trilinear interpolation)
        uvw = [(pos[i] - config.bbmin[i]) / (config.bbmax[i] - config.bbmin[i]) for i in range(3)]
        uvw = [max(0.0, min(1.0, u)) for u in uvw]

        # Sample density
        if config.bVolume4D:
            d1 = sample_3d_texture(uvw, volume_texture)
            d2 = sample_3d_texture(uvw, config.volume_texture_4d_next)
            density = d1 * (1.0 - config.interpolationFactor) + d2 * config.interpolationFactor
        else:
            density = sample_3d_texture(uvw, volume_texture)

        # Density threshold check
        if density > 0.01:
            # Color mapping
            norm_density = clamp((density - config.scalar_min) / (config.scalar_max - config.scalar_min), 0.0, 1.0)
            s = sample_color_map(norm_density, color_map) # s is [r, g, b, extinction]

            # Compositing
            alpha = 1.0 - exp(-s[3] * dt)
            contribution = [transmittance * s[0] * alpha, transmittance * s[1] * alpha, transmittance * s[2] * alpha]
            accumulated_color = [accumulated_color[i] + contribution[i] for i in range(3)]
            transmittance *= (1.0 - alpha)

            # First hit storage
            if not first_hit:
                first_hit = True
                # store_position_depth(pos, t) # Handled externally

            # Early termination
            if 1.0 - transmittance > 0.95:
                break

            prev_density = density

            # Step size adaptation (General Variant from S2)
            d_density = density - prev_density
            new_step = get_step(d_density, transmittance, density, config.step_scale)
            dt = (dt + new_step) * 0.5

        else:
            # Empty space handling
            if config.use_cloud_steps:
                cur_threshold -= 1
                if cur_threshold <= 0:
                    dt = cur_coarse_step
            else:
                # Increase step size in empty space
                dt = min(dt * 2.0, config.max_step)

        # Advance
        t += dt

    # Post-processing
    accumulated_color = [pow(c, 1.0/2.2) for c in accumulated_color]
    final_alpha = 1.0 - transmittance
    final_color = [accumulated_color[i] * config.illumination_scale for i in range(3)]

    # Temporal accumulation
    if config.accumulation and not config.camera_moved:
        last_color = config.last_frame_color
        final_color = [last_color[i] * (1.0 - final_alpha) + final_color[i] * final_alpha for i in range(3)]

    return final_color, final_alpha

# Standard helper operations [G]
def clamp(x, lo, hi):
    return max(lo, min(hi, x))

def sample_3d_texture(uvw, tex):
    # Trilinear interpolation of 3D volume texture at normalized coordinates uvw
    return tex.trilinear_sample(uvw)

def sample_color_map(coord, cmap):
    # 1D color/transfer function lookup
    return cmap.sample(coord)

def get_step(d_density, transmittance, density, step_scale):
    # Standard adaptive step function based on density gradient and transmittance
    return step_scale * (1.0 + abs(d_density)) * transmittance
```

## Subtopics

### [Screen-Space Ray Casting with Adaptive Step Size](16661-screen-space-ray-casting-with-adaptive-step-size.md)
Screen-Space Ray Casting with Adaptive Step Size is a pixel shader technique that renders volumetric scattering by casting rays from the camera through each pixel into a 3D texture, accumulating in-scattered radiance using an adaptive step size based on density changes and transmittance. It distinguishes itself from fixed-step ray marching by dynamically adjusting the sampling interval to maintain visual fidelity in high-density regions while skipping empty space, and it operates entirely in screen space without requiring explicit geometry intersection beyond depth buffer occlusion.

### Raymarching with Adaptive Step Size and Early Termination
A volumetric rendering technique for clouds that uses raymarching with single scattering toward the sun, optimized with adaptive step sizing and early termination to maintain interactive frame rates [S5, unofficial]. Adaptive step sizing increases step length with ray distance and in empty regions; upon entering a cloud volume, the ray backsteps and resumes with finer steps [S5, unofficial]. Early termination halts the marching loop immediately when accumulated transmittance falls below a predefined threshold, skipping further sampling in that region [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples per ray. Adaptive step size reduces the effective $N$ by skipping empty space, but increases overhead per step due to density gradient calculations and conditional branching [S1].
*   **Memory Complexity:** $O(1)$ per ray, plus storage for volume textures and auxiliary buffers (noise, depth, motion vectors) [S3, unofficial].
*   **Performance:** Generates high-quality images with SSIM of 0.95 compared to source, with significant speedup over linear sampling [S1]. Interactive frame rates are achievable for cloud rendering with early termination and adaptive steps [S5, unofficial].
*   **Strengths:** Reduces computational cost in empty regions; maintains quality in dense areas; supports temporal accumulation for denoising [S1][S3, unofficial].
*   **Limitations:** Branch divergence in GPU shaders due to variable step sizes; requires careful tuning of thresholds and multipliers; 4D volumes require double the texture memory [S1][S3, unofficial].
*   **When to use:** Real-time volume rendering where performance is critical and density fields have large empty regions (e.g., clouds, smoke) [S1][S4, unofficial].
*   **When not to use:** Uniform density fields where adaptive steps provide no benefit; scenes requiring precise surface intersection without approximation [S2, S5, unofficial].

## Background
Adaptive sampling for volume rendering was proposed to tackle the excessive computational cost of GPU-based 3-D volume rendering, making a trade-off between performance and rendering quality [S1]. The method reduces the number of sampling points through a non-linear sampling function, inspired by photography where far objects are vague and close objects are clear [S1]. It is implemented in various projects including DDVR (Direct Volume Renderer) which supports 3D and 4D volumes with temporal accumulation and denoising [S3, unofficial], and cloud renderers using procedural noise [S4, S5, unofficial]. The technique builds on [Adaptive Step Size Control](9843-adaptive-step-size-control.md) principles [S1][S2, S3, S4, S5, unofficial].

## Sources
[S1] Adaptive Sampling for GPU-based 3-D Volume Rendering, https://doi.org/10.1145/3285996.3286002, fetched 2026-11-02
[S2] StarsX/FluidX12, https://github.com/StarsX/FluidX12, fetched 2026-10-26 (unofficial)
[S3] AoGao-Kedoka/DDVR, https://github.com/AoGao-Kedoka/DDVR, fetched 2026-10-26 (unofficial)
[S4] Depersonalizc/volumetric-clouds, https://github.com/Depersonalizc/volumetric-clouds, fetched 2026-10-27 (unofficial)
[S5] Klark007/OpenGL-Cloud-Renderer, https://github.com/Klark007/OpenGL-Cloud-Renderer, fetched 2026-10-27 (unofficial)
