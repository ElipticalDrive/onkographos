# Adaptive Step Size Ray Marching

Also called: DDVR, DDVR Raymarching Shader, Ray Marching with Adaptive Step Size and Temporal Accumulation, Adaptive Step Size Volume Rendering

## Overview
Adaptive Step Size Ray Marching is a GPU-based direct volume rendering technique that reduces computational cost by varying the ray marching step size based on local density and distance, rather than using a fixed interval [S1][S2, unofficial]. It distinguishes itself from linear sampling by simulating photographic depth of field—using fine steps for clear, dense regions and coarse steps for empty space—while often incorporating temporal accumulation and noise-based step jittering to maintain visual quality at low sample counts [S1][S2, unofficial]. It builds on [Adaptive Step Size Control](9843-adaptive-step-size-control.md).

## How it works
### Inputs and Outputs
**Inputs:**
*   Camera state: Position, view/projection matrices, movement flag (`cam.moved`) [S2, unofficial].
*   Volume data: 3D or 4D (time-varying) scalar field textures [S2, unofficial].
*   Configuration: Bounding box (`bbmin`, `bbmax`), base `stepSize`, `scalarMinMax`, `illuminationScale`, `rotationAngles` [S2, unofficial].
*   Per-pixel state: `noiseImage` storing current step size (`.r`) and ray offset (`.g`) [S2, unofficial].
*   Previous frame data: `outputImage` (for accumulation), `positionDepthBufferImage` (for reprojection) [S2, unofficial].

**Outputs:**
*   `outputImage`: Final RGBA color [S2, unofficial].
*   `positionDepthBufferImage`: World position and depth of the first hit [S2, unofficial].
*   `motionVectorImage`: UV-space motion vectors (optional) [S2, unofficial].
*   `noiseImage`: Updated step size and offset for the next frame [S2, unofficial].

### Data Structures
**Uniform Block (set=0, binding=0):**
*   `dimensions`: `vec3` (volume dimensions) [S2, unofficial].
*   `bbmin`, `bbmax`: `vec3` (bounding box) [S2, unofficial].
*   `rotationAngles`: `vec2` (x, y rotation for volume) [S2, unofficial].
*   `scalarMinMax`: `vec2` (min/max density for normalization) [S2, unofficial].
*   `stepSize`: `float` (base step size) [S2, unofficial].
*   `illuminationScale`: `float` [S2, unofficial].
*   `ellapsedTime`: `float` [S2, unofficial].
*   `accumulation`: `int` (1 or 0) [S2, unofficial].
*   `motionVectorCalc`: `int` (1 or 0) [S2, unofficial].
*   `bVolume4D`: `int` (1 or 0) [S2, unofficial].
*   `interpolationFactor`: `float` (for 4D mixing) [S2, unofficial].
*   `pingpong4D`: `int` (0 or 1) [S2, unofficial].

**Textures/Images:**
*   `noiseImage` (`image2D`, `rg32f`): `.r` = current step size (0 if default), `.g` = ray offset [S2, unofficial].
*   `volumeTexture` (`sampler3D`): 3D density data [S2, unofficial].
*   `volumeTexture4DNextFrame` (`sampler3D`): Next frame density data (if 4D) [S2, unofficial].
*   `colorMapping` (`sampler1D`): Transfer function [S2, unofficial].
*   `outputImage` (`image2D`, `rgba8`): Final color [S2, unofficial].
*   `positionDepthBufferImage` (`image2D`, `rgba32f`): `.xyz` = world position, `.w` = depth `t` [S2, unofficial].
*   `motionVectorImage` (`image2D`, `rgba8`): `.xy` = motion vector, `.z`=0, `.w`=1 [S2, unofficial].
*   `previousCam` buffer: Camera state from previous frame [S2, unofficial].

### Numbered Steps
1.  **Ray Initialization:** Construct ray origin from camera position and direction from UV coordinates, projection, and view matrices [S2, unofficial]. Perform ray-box intersection against `bbmin`/`bbmax` to find entry (`tEntry`) and exit (`tExit`) distances [S2, unofficial]. If no intersection, output black color and zero depth, then return [S2, unofficial].
2.  **State Initialization:** Initialize `transmittance = 1.0`, `accumulatedColor = vec3(0.0)`, `firstHit = false` [S2, unofficial]. Calculate `totalDst = tExit - tEntry` [S3, unofficial]. Set `curFineStepSize = min(STEPSIZE_FINE, totalDst / MIN_NUM_FINE_STEPS)` [S3, unofficial]. Set `curCoarseStepSize = curFineStepSize * COARSE_STEPSIZE_MULTIPLIER` [S3, unofficial]. Set `curThreshold = MAX_NUM_MISSED_STEPS` [S3, unofficial]. Load `rayOffset` from `noiseImage[pixelCoord].g` [S2, unofficial].
3.  **Step Size Determination:** Load `noiseOffset` from `noiseImage[pixelCoord]` [S2, unofficial]. If `noiseOffset.r` is 0, use base `stepSize`; otherwise, use `noiseOffset.r` as `m_stepSize` [S2, unofficial]. Calculate `stepDir = rayDir * m_stepSize` [S2, unofficial].
4.  **Ray Marching Loop:** Iterate `t` from `tEntry + rayOffset` to `tExit` with step `m_stepSize` [S2, unofficial].
    *   **Coordinate Calculation:** Calculate UVW coordinates relative to the bounding box, applying `rotationAngles` [S2, unofficial].
    *   **Density Sampling:**
        *   If `bVolume4D == 1`: Sample `density1` from `volumeTexture` and `density2` from `volumeTexture4DNextFrame` using `pingpong4D` flag. Mix using `interpolationFactor` [S2, unofficial].
        *   If `bVolume4D == 0`: Sample `density` directly from `volumeTexture` [S2, unofficial].
    *   **Density Threshold Check:** If `density > 0.01`, proceed; otherwise, decrement `curThreshold` by 1 [S3, unofficial].
    *   **Color Mapping:** Calculate `colorMappingCoord = clamp((density - scalarMinMax.x) / (scalarMinMax.y - scalarMinMax.x), 0.0, 1.0)` [S2, unofficial]. Sample `colorMapping` texture to get `s` (vec4) [S2, unofficial].
    *   **Compositing:** Extract `extinction = s.a` [S2, unofficial]. Calculate `alpha = 1.0 - exp(-extinction * m_stepSize)` [S2, unofficial]. Calculate `contribution = transmittance * s.rgb * alpha` [S2, unofficial]. Update `accumulatedColor += contribution` [S2, unofficial]. Update `transmittance *= (1.0 - alpha)` [S2, unofficial].
    *   **First Hit Storage:** If `!firstHit` and `alpha > 0.0`, set `firstHit = true` and store `rayOrigin + rayDir * t` and `t` in `positionDepthBufferImage` [S2, unofficial].
    *   **Early Termination:** If `1.0 - transmittance > 0.95` (i.e., `transmittance < 0.05`), break loop [S2, unofficial].
    *   **Advance Position:** `pos += stepDir` [S2, unofficial].
5.  **Post-Processing:** Apply gamma correction: `accumulatedColor.rgb = pow(accumulatedColor.rgb, vec3(1.0 / 2.2))` [S2, unofficial]. Calculate `finalAlpha = 1.0 - transmittance` [S2, unofficial]. Calculate `finalColor = vec4(accumulatedColor, finalAlpha) * illuminationScale` [S2, unofficial].
6.  **Temporal Accumulation:** If `accumulation == 1` and `!cam.moved`, blend `finalColor` with `lastFrameColor` (from `outputImage`) using `finalAlpha` as weight: `finalColor.rgb = lastFrameColor.rgb * (1.0 - finalAlpha) + finalColor.rgb * finalAlpha` [S2, unofficial]. If `cam.moved`, reset `noiseImage.g` to 0 [S2, unofficial].
7.  **Motion Vectors (Optional):** If `motionVectorCalc == 1`, load position from `positionDepthBufferImage`. If position is zero, store zero vector. Otherwise, reproject position to previous frame coordinates using `previousCam` and calculate UV motion vector diff, storing in `motionVectorImage` [S2, unofficial].
8.  **Update Noise/Output:** Store `finalColor` to `outputImage` [S2, unofficial]. Update `noiseImage`: If `accumulation == 1`, call `updateNoiseOffset` (increment `g` by `(1 / m_stepSize * 10) * 0.00001` if `g < m_stepSize`, else reset) [S2, unofficial]. If `!accumulation`, store `noiseOffset.r` and reset `g` to 0 [S2, unofficial].

### Formulas
*   **Ray Position:** $\mathbf{p}_{i+1} = \mathbf{p}_i + \Delta t_i \mathbf{d}$ [Parent].
*   **Step Size:** $\Delta t_i = \text{noiseImage}[\text{pixelCoord}].r$ if non-zero, else $\text{stepSize}$ [S2, unofficial].
*   **Alpha:** $\alpha = 1.0 - \exp(-\text{extinction} \cdot \Delta t_i)$ [S2, unofficial].
*   **Transmittance Update:** $T_{i+1} = T_i \cdot (1.0 - \alpha)$ [S2, unofficial].
*   **Color Contribution:** $C_{contrib} = T_i \cdot \mathbf{s}.rgb \cdot \alpha$ [S2, unofficial].
*   **Accumulated Color:** $\mathbf{C}_{acc} += C_{contrib}$ [S2, unofficial].
*   **Temporal Blend:** $\mathbf{C}_{final} = \mathbf{C}_{prev} \cdot (1.0 - \alpha_{final}) + \mathbf{C}_{curr} \cdot \alpha_{final}$ [S2, unofficial].

### Parameters
*   `stepSize`: Base step size [S2, unofficial].
*   `STEPSIZE_FINE`: Max fine step size [S3, unofficial].
*   `COARSE_STEPSIZE_MULTIPLIER`: Multiplier for coarse steps [S3, unofficial].
*   `MAX_NUM_MISSED_STEPS`: Threshold for switching to coarse steps [S3, unofficial].
*   `EARLY_STOP_THRESHOLD`: Transmittance threshold (0.05) for early termination [S3, unofficial].
*   `scalarMinMax`: Density normalization range [S2, unofficial].
*   `illuminationScale`: Final color multiplier [S2, unofficial].
*   `accumulation`: Enable temporal accumulation [S2, unofficial].
*   `bVolume4D`: Enable 4D volume support [S2, unofficial].
*   `interpolationFactor`: 4D frame mixing factor [S2, unofficial].
*   `pingpong4D`: 4D texture swap flag [S2, unofficial].

### Edge Cases and Pitfalls
*   **Camera Movement:** If `cam.moved` is true, `positionDepthBufferImage` and `noiseImage.g` must be reset to zero to prevent accumulation artifacts and invalid reprojection [S2, unofficial].
*   **No Intersection:** If ray-box intersection fails, output black color and zero depth [S2, unofficial].
*   **Zero Position:** If `positionDepthBufferImage.xyz` is zero, motion vector is set to zero to avoid division by zero [S2, unofficial].
*   **Color Banding:** Minimized by applying random offset to ray start point based on fragment coordinates [S3, unofficial].
*   **Transmittance Approximation:** Some implementations use Taylor approximation for $\exp(-\text{density} \cdot \text{cloudLightAbsorptionMult} \cdot dt)$ instead of exact exponential [S3, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
EARLY_STOP_THRESHOLD = 0.05
STEPSIZE_FINE = 0.01 # Example value
MIN_NUM_FINE_STEPS = 100 # Example value
COARSE_STEPSIZE_MULTIPLIER = 4.0 # Example value
MAX_NUM_MISSED_STEPS = 10 # Example value

def adaptive_step_size_ray_marching(pixel_coord, camera, volume_data, noise_image, output_image, position_depth_buffer, motion_vector_image, config):
    # 1. Ray Initialization
    ray_origin = camera.transform[3].xyz
    ray_dir = normalize(calculate_ray_direction(pixel_coord, camera.proj, camera.view))
    
    t_entry, t_exit = ray_box_intersection(ray_origin, ray_dir, config.bbmin, config.bbmax)
    if t_entry is None:
        output_image[pixel_coord] = vec4(0.0, 0.0, 0.0, 1.0)
        position_depth_buffer[pixel_coord] = vec4(0.0, 0.0, 0.0, 0.0)
        return

    # 2. State Initialization
    transmittance = 1.0
    accumulated_color = vec3(0.0)
    first_hit = False
    total_dst = t_exit - t_entry
    cur_fine_step_size = min(STEPSIZE_FINE, total_dst / MIN_NUM_FINE_STEPS)
    cur_coarse_step_size = cur_fine_step_size * COARSE_STEPSIZE_MULTIPLIER
    cur_threshold = MAX_NUM_MISSED_STEPS
    ray_offset = noise_image[pixel_coord].g
    
    # 3. Step Size Determination
    noise_offset = noise_image[pixel_coord]
    if noise_offset.r == 0.0:
        m_step_size = config.stepSize
    else:
        m_step_size = noise_offset.r
        
    step_dir = ray_dir * m_step_size
    pos = ray_origin + ray_dir * t_entry
    
    # 4. Ray Marching Loop
    t = t_entry + ray_offset
    while t < t_exit:
        # Coordinate Calculation
        uvw = calculate_uvw(pos, config.bbmin, config.bbmax, config.rotationAngles)
        
        # Density Sampling
        if config.bVolume4D:
            density1 = sample_volume(volume_data.volumeTexture, uvw)
            density2 = sample_volume(volume_data.volumeTexture4DNextFrame, uvw)
            density = mix(density1, density2, config.interpolationFactor)
        else:
            density = sample_volume(volume_data.volumeTexture, uvw)
            
        # Density Threshold Check
        if density <= 0.0:
            cur_threshold -= 1
            # Optional: Switch to coarse step if threshold exhausted
            # if cur_threshold <= 0: m_step_size = cur_coarse_step_size; step_dir = ray_dir * m_step_size; cur_threshold = MAX_NUM_MISSED_STEPS
            
        if density > 0.01:
            # Color Mapping
            color_map_coord = clamp((density - config.scalarMinMax.x) / (config.scalarMinMax.y - config.scalarMinMax.x), 0.0, 1.0)
            s = sample_color_mapping(config.colorMapping, color_map_coord)
            
            # Compositing
            extinction = s.a
            alpha = 1.0 - exp(-extinction * m_step_size)
            contribution = transmittance * s.rgb * alpha
            accumulated_color += contribution
            transmittance *= (1.0 - alpha)
            
            # First Hit Storage
            if not first_hit and alpha > 0.0:
                first_hit = True
                position_depth_buffer[pixel_coord] = vec4(pos, t)
                
            # Early Termination
            if 1.0 - transmittance > EARLY_STOP_THRESHOLD:
                break
                
        # Advance Position
        pos += step_dir
        t += m_step_size
        
    # 5. Post-Processing
    accumulated_color.rgb = pow(accumulated_color.rgb, vec3(1.0 / 2.2))
    final_alpha = 1.0 - transmittance
    final_color = vec4(accumulated_color, final_alpha) * config.illuminationScale
    
    # 6. Temporal Accumulation
    if config.accumulation and not camera.moved:
        last_frame_color = output_image[pixel_coord]
        final_color.rgb = last_frame_color.rgb * (1.0 - final_alpha) + final_color.rgb * final_alpha
        final_color.a = last_frame_color.a * (1.0 - final_alpha) + final_color.a * final_alpha
    elif config.accumulation and camera.moved:
        noise_image[pixel_coord].g = 0.0
        
    # 7. Motion Vectors (Optional)
    if config.motionVectorCalc:
        pos_depth = position_depth_buffer[pixel_coord]
        if pos_depth.xyz == vec3(0.0):
            motion_vector_image[pixel_coord] = vec4(0.0, 0.0, 0.0, 1.0)
        else:
            # Reprojection logic using previousCam
            prev_pos = reproject_position(pos_depth, camera.previousCam, camera)
            motion_vec = calculate_motion_vector_uv(pos_depth, prev_pos, pixel_coord)
            motion_vector_image[pixel_coord] = vec4(motion_vec, 0.0, 1.0)
            
    # 8. Update Noise/Output
    output_image[pixel_coord] = final_color
    
    if config.accumulation:
        step_offset = noise_image[pixel_coord].g
        if step_offset < m_step_size:
            noise_image[pixel_coord].g += (1.0 / m_step_size * 10.0) * 0.00001
        else:
            noise_image[pixel_coord].g = 0.0
    else:
        noise_image[pixel_coord].g = 0.0
```

## Subtopics

### Dynamic Sampling Step Adjustment
Adjusts step size based on local density gradients or error estimates, often using a separate error metric. Choose when high-frequency detail preservation is critical. [Link](2442-dynamic-sampling-step-adjustment.md)

### Adaptive Step Size Control
The general framework for determining step size based on ray state. Choose as the foundational strategy for any adaptive method. [Link](9843-adaptive-step-size-control.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples per ray. Adaptive step size reduces $N$ significantly in empty regions, leading to faster rendering compared to linear sampling [S1].
*   **Memory Complexity:** $O(1)$ per ray, plus storage for `noiseImage`, `positionDepthBufferImage`, and `motionVectorImage` [S2, unofficial].
*   **Measured Performance:** Generates images with SSIM of 0.95 compared to source, indicating high quality [S1]. Reduces computational cost by varying step size [S2, unofficial].
*   **Scaling:** Scales well on GPUs due to parallelizable ray casting [S1]. 4D support adds minimal overhead with texture interpolation [S2, unofficial].
*   **Strengths:** High efficiency in computation, trade-off between performance and quality, supports temporal accumulation for denoising [S1][S2, unofficial].
*   **Limitations:** Requires additional buffers for accumulation and motion vectors. Noise-based step size may introduce artifacts if not managed [S2, unofficial].
*   **When to Use:** Real-time direct volume rendering where performance is critical, especially for large or sparse volumes [S1][S2, unofficial].
*   **When Not to Use:** When absolute geometric precision is required without temporal accumulation, or when hardware lacks support for required image formats [S2, unofficial].

## Background
Originated from the need to reduce excessive computational cost in GPU-based 3-D volume rendering [S1]. The method was proposed to tackle the dilemma of fixed step sizes in linear sampling, where small steps ensure quality but reduce efficiency, and large steps ensure efficiency but reduce quality [S1]. It was implemented in the DDVR renderer [S2, unofficial] and used in volumetric cloud rendering [S3, unofficial]. The technique simulates photographic depth of field, where far objects are vague and close objects are clear [S1].

## Sources
[S1] Adaptive Sampling for GPU-based 3-D Volume Rendering, https://doi.org/10.1145/3285996.3286002, fetched 2026-11-02
[S2] AoGao-Kedoka/DDVR, https://github.com/AoGao-Kedoka/DDVR, fetched 2026-10-26 (unofficial)
[S3] Depersonalizc/volumetric-clouds, https://github.com/Depersonalizc/volumetric-clouds, fetched 2026-10-27 (unofficial)
