# Dynamic Sampling Step Adjustment

Also called: Adaptive Step Doubling, Empty Space Step Doubling

## Overview
Dynamic Sampling Step Adjustment is a ray marching optimization technique that increases the sampling step size when traversing empty or low-density regions of a volume, thereby reducing the number of texture lookups and shading computations [S1, unofficial]. It solves the problem of redundant sampling in sparse volumes where uniform step sizes waste computational resources on empty space [S1, unofficial]. Unlike [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md), which relies on geometric bounds or SDF values to guarantee no overshooting, this method relies on statistical emptiness (zero density) detected via hierarchical mipmaps, trading strict geometric safety for performance gains in specific content types like volumetric clouds [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, volume texture with mipmaps, initial step size $\Delta s_0$, initial mipmap level $m_0$.
*   **Output:** Transmittance and accumulated color along the ray.

**Data Structures:**
*   **Volume Texture:** A 3D texture with multiple mip levels. Lower mip levels represent coarser, downsampled versions of the density field.
*   **State Variables:**
    *   `current_step`: Current step size $\Delta s$.
    *   `current_mip`: Current mipmap level index $m$.
    *   `ray_pos`: Current position $\mathbf{p}$ along the ray.

**Algorithm Steps:**
1.  **Initialization:** Set $\Delta s = \Delta s_0$, $m = m_0$, and $\mathbf{p} = \mathbf{o}$.
2.  **Sample Density:** At $\mathbf{p}$, sample the density $\rho$ from the volume texture at mipmap level $m$.
3.  **Check Emptiness:**
    *   If $\rho \approx 0$ (empty) for a consecutive number of samples (typically 1 or more, depending on implementation [S1, unofficial]):
        *   Double the step size: $\Delta s \leftarrow 2 \cdot \Delta s$.
        *   Increase the mipmap level: $m \leftarrow m + 1$ (if $m < m_{max}$).
        *   Advance the ray: $\mathbf{p} \leftarrow \mathbf{p} + \Delta s \cdot \mathbf{d}$.
        *   Repeat Step 2.
    *   If $\rho > 0$ (non-empty):
        *   Step back one step: $\mathbf{p} \leftarrow \mathbf{p} - \Delta s \cdot \mathbf{d}$.
        *   Restore step size: $\Delta s \leftarrow \Delta s / 2$.
        *   Restore mipmap level: $m \leftarrow m - 1$.
        *   Proceed with standard rendering/compositing at the restored parameters.
4.  **Termination:** Stop when the ray exits the volume or transmittance drops below a threshold.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_step_adjustment(ray_origin, ray_direction, volume_texture, initial_step, initial_mip):
    step_size = initial_step
    mip_level = initial_mip
    current_pos = ray_origin
    
    # Constants
    EMPTY_THRESHOLD = 1e-6
    MAX_MIP = volume_texture.num_mip_levels - 1
    
    while current_pos is inside_volume(current_pos):
        # Sample density at current position and mip level
        density = volume_texture.sample(current_pos, mip_level)
        
        if density < EMPTY_THRESHOLD:
            # Region is empty: accelerate
            step_size *= 2.0
            if mip_level < MAX_MIP:
                mip_level += 1
            
            # Advance ray
            current_pos += ray_direction * step_size
        else:
            # Region is non-empty: retreat and restore
            current_pos -= ray_direction * step_size
            step_size /= 2.0
            if mip_level > 0:
                mip_level -= 1
            
            # Perform compositing for this step (or subsequent steps)
            # Note: The algorithm restores state before processing the non-empty segment
            break # Or continue to process the non-empty segment with restored params
            
    return accumulated_color
```

**Parameters:**
*   `initial_step`: The base step size for the finest mipmap level. Typical values depend on volume resolution and view distance.
*   `EMPTY_THRESHOLD`: The density value below which a sample is considered empty. [G] This value should be slightly above zero to account for floating-point noise, e.g., $10^{-6}$.
*   `MAX_MIP`: The maximum mipmap level allowed.

**Edge Cases and Pitfalls:**
*   **Artifacts:** The method may cause visual artifacts if the step size becomes too large relative to the feature size in the non-empty regions, leading to aliasing or missed details [S1, unofficial].
*   **Precision:** Doubling the step size can lead to large jumps, potentially skipping thin structures if the emptiness check is not robust.
*   **Boundary Conditions:** If the ray exits the volume while accelerating, the termination condition must be checked carefully to avoid out-of-bounds texture access.

## Subtopics

### [Dynamic Texture Slice Resampling](6101-dynamic-texture-slice-resampling.md)
Dynamic Texture Slice Resampling is a technique for volume rendering that dynamically updates slicing steps according to viewing directions to maintain a constant step size during the Direct Volume Rendering (DVR) process. It solves the problem of striping artifacts caused by non-uniform sampling steps, which occur when fixed step sizes result in varying spatial resolution across the volume due to perspective projection. Unlike methods that rely on post-processing upsampling or fixed-step approximations, this approach maintains high rendering performance by adapting the sampling density to the viewing geometry in real-time.

### Gradient-driven Sampling
Gradient-driven sampling is a strategy that refines step sizes according to the magnitude of the scalar field gradient $|\nabla s|$, which indicates local feature sharpness; smaller steps are taken near high-gradient regions resembling surfaces to capture transitions accurately [S2]. It dynamically adjusts the step size based on the gradient direction and a threshold, bounded by minimum and maximum distances to prevent under- or over-sampling [S2]. This method enhances detail preservation in boundary-heavy volumes, such as CT angiography data, while coarsening traversal in smooth interiors, achieving efficiency gains of 2-3x over uniform sampling without aliasing artifacts [S2].

### Dynamic Adaptive Sampling Ray-Casting
Dynamic Adaptive Sampling Ray-Casting is an improved volume rendering algorithm designed to address slow rendering speed and low sampling efficiency in 3D reconstruction of medical images by dynamically adjusting sampling intervals and using a new rendering operator [S3]. It uses the central difference gradient method to obtain the corresponding sampling interval dynamically according to the different sampling points [S3]. It applies a new rendering operator based on the color value and opacity changes before and after the ray enters the volume element, and the resistance luminosity [S3]. On the ISBI2018 dataset, the ray-casting DAS algorithm reduces and avoids artifact phenomena in CT image rendering and improves image visibility compared to Francisco Sans' algorithm, Roba Binyahibs' algorithm, Feiniu Yuan's algorithm, Leonardo's algorithm, and Yan Zhang's algorithm [S3]. Compared with other algorithms, this method achieves faster rendering speed while ensuring the quality of the generated image [S3].

### Geometry Sampling Adjustment
Geometry Sampling Adjustment is a method to adjust the maximum number of geometry samples per streamline based on a geometry sampling factor [S4, unofficial]. It sets `m_maxGeometrySamples` to `m_maxSamples / m_geometrySampling + 1` [S4, unofficial]. The parameter `m_maxSamples` is the base maximum number of samples per streamline, and `m_geometrySampling` is a factor used to reduce the number of geometry samples [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples taken. However, the effective $N$ is significantly reduced in empty regions due to step doubling.
*   **Memory Complexity:** $O(V)$ where $V$ is the size of the volume texture, including all mip levels.
*   **Performance:** Provides small performance gains in sparse volumes [S1, unofficial]. Efficiency gains of 2-3x have been reported for gradient-driven variants in boundary-heavy volumes [S2].
*   **Strengths:** Simple to implement; effective for volumes with large empty regions (e.g., clouds, fog).
*   **Limitations:** May cause artifacts if the step size increases too aggressively [S1, unofficial]. Not suitable for volumes where thin, high-frequency details are critical and not captured by the mipmap hierarchy.
*   **When to use:** Volumetric clouds, fog, or other sparse volumes where empty space dominates.
*   **When not to use:** Dense volumes with fine details throughout, or when strict geometric accuracy is required without post-processing refinement.

## Background
Dynamic Sampling Step Adjustment, also known as Adaptive Step Doubling or Empty Space Step Doubling, is a technique used in real-time volume rendering, particularly for volumetric clouds and atmospheric effects [S1, unofficial]. It leverages the hierarchical nature of mipmapped textures to skip empty space efficiently. The method is often discussed in the context of game engine implementations, such as those for rendering volumetric clouds in modern graphics pipelines [S1, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] Volume ray casting — Grokipedia, https://grokipedia.com/page/Volume_ray_casting, fetched 2026-10-26
[S3] (PDF) A Comparison between GPU-based Volume Ray Casting ..., https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S4] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
