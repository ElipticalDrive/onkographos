# Dynamic Sampling Step Adjustment

Also called: Adaptive Step Doubling, Empty Space Step Doubling

## Overview
Dynamic Sampling Step Adjustment is a ray marching optimization technique that increases the integration step size when traversing empty or low-density regions of a volume, and restores it upon encountering data. It solves the problem of redundant sampling in sparse volumes, such as atmospheric clouds or low-density medical scans, by skipping large empty intervals. It sets itself apart from [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by prioritizing density-based skipping over geometric distance-field safety constraints, and from [Adaptive Step Size Ray Marching](2312-adaptive-step-size-ray-marching.md) by using a discrete doubling/halving logic rather than continuous gradient-based scaling.

## How it works
**Inputs:**
*   Ray origin $\mathbf{o}$ and direction $\mathbf{d}$ (normalized).
*   Volume texture $V$ with associated mipmaps.
*   Initial step size $\Delta t_{base}$ (scalar).
*   Initial mipmap level $m_{base}$ (integer, typically 0).
*   Empty threshold $\tau$ (scalar, typically 0.0 for density).
*   Maximum empty count $N_{max}$ (integer, optional guard against infinite skipping).

**Outputs:**
*   Accumulated color and opacity along the ray.
*   Updated step size $\Delta t$ and mipmap level $m$ for the current ray segment.

**Data Structures:**
*   `step_size`: Float, current integration step length.
*   `mip_offset`: Integer, current mipmap level index.
*   `empty_count`: Integer, consecutive samples with density $\le \tau$.

**Algorithm Steps:**
1.  Initialize $\Delta t = \Delta t_{base}$, $m = m_{base}$, and `empty_count` = 0.
2.  Sample the volume at current position $\mathbf{p}$ using texture $V_m$ (mipmap level $m$). Let density be $\rho = V_m(\mathbf{p})$.
3.  **Check for Empty Space:**
    *   If $\rho \le \tau$:
        *   Increment `empty_count`.
        *   If `empty_count` > 1 (i.e., this is the second or more consecutive empty sample):
            *   Double the step size: $\Delta t \leftarrow 2 \cdot \Delta t$.
            *   Increase mipmap level: $m \leftarrow m + 1$.
            *   Clamp $m$ to the maximum available mipmap level.
        *   Advance position: $\mathbf{p} \leftarrow \mathbf{p} + \Delta t \cdot \mathbf{d}$.
        *   Go to Step 2.
    *   If $\rho > \tau$:
        *   **Step Back:** Move position back by one base step to ensure the surface/feature is captured accurately: $\mathbf{p} \leftarrow \mathbf{p} - \Delta t_{base} \cdot \mathbf{d}$. [S1, unofficial]
        *   **Restore State:** Reset $\Delta t = \Delta t_{base}$ and $m = m_{base}$. [S1, unofficial]
        *   Reset `empty_count` = 0.
        *   Proceed with standard rendering/sampling at the restored position.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_sampling_step_adjustment(ray_origin, ray_dir, volume_texture, base_step, base_mip, empty_threshold):
    step_size = base_step
    mip_level = base_mip
    empty_count = 0
    
    # Assume max_mip is known from volume texture metadata
    max_mip = volume_texture.num_mip_levels - 1
    
    while ray_t < max_distance:
        # Sample current position
        density = volume_texture.sample_mip(ray_origin + ray_t * ray_dir, mip_level)
        
        if density <= empty_threshold:
            empty_count += 1
            
            # Double step size and mip level after the first empty sample
            # The first empty sample is processed with current step; 
            # subsequent ones trigger acceleration.
            if empty_count > 1:
                step_size *= 2.0
                if mip_level < max_mip:
                    mip_level += 1
            
            ray_t += step_size
        else:
            # Density found: step back to capture the transition accurately
            ray_t -= step_size # Or specifically base_step depending on implementation nuance [S1, unofficial]
            
            # Restore original parameters
            step_size = base_step
            mip_level = base_mip
            empty_count = 0
            
            # Perform standard volume integration step here
            # ...
            break # Or continue with normal small steps until next empty region
            
    return accumulated_color
```

**Edge Cases and Pitfalls:**
*   **Artifacts:** The method may cause visual artifacts, particularly "popping" or aliasing, if the step doubling skips over thin features that are aliased at the coarser mipmap level [S1, unofficial].
*   **Precision:** Floating-point precision errors in ray position accumulation can cause the "step back" to land slightly off the true boundary, potentially missing thin structures if $\Delta t_{base}$ is not sufficiently small relative to feature size.
*   **Boundary Conditions:** If the ray exits the volume bounds while skipping, the step back might land outside the valid volume range, requiring boundary checks.

## Subtopics

### [Gradient-driven Sampling](5281-gradient-driven-sampling.md)
Gradient-driven sampling is a dynamic step size adjustment method for distance-field ray marching that varies the integration step length based on the local magnitude of the scalar field gradient. It solves the problem of balancing rendering fidelity and performance in volumes with sharp boundaries (e.g., CT angiography) by taking smaller steps near high-gradient regions to capture transitions accurately and larger steps in smooth interiors. It sets itself apart from uniform sampling by adapting to local feature sharpness and from hierarchical sampling by using a continuous gradient magnitude threshold rather than discrete variance-based subdivision.

### [Dynamic Texture Slice Resampling](6101-dynamic-texture-slice-resampling.md)
Dynamic Texture Slice Resampling is a technique for volume rendering that dynamically updates slicing steps according to viewing directions to maintain a constant step size during the Direct Volume Rendering (DVR) process. It solves the problem of striping artifacts caused by non-uniform sampling steps, which occur when fixed step sizes result in varying spatial resolution across the volume due to perspective projection. Unlike methods that rely on post-processing upsampling or fixed-step approximations, this approach maintains high rendering performance by adapting the sampling density to the viewing geometry in real-time.

### Dynamic Adaptive Sampling Ray-Casting
Dynamic Adaptive Sampling Ray-Casting is an improved volume rendering algorithm designed to address slow rendering speed and low sampling efficiency in 3D reconstruction of medical images by dynamically adjusting sampling intervals and using a new rendering operator [S2]. It uses the central difference gradient method to obtain the corresponding sampling interval dynamically according to the different sampling points. It applies a new rendering operator based on the color value and opacity changes before and after the ray enters the volume element, and the resistance luminosity. On the ISBI2018 dataset, this algorithm reduces and avoids artifact phenomena in CT image rendering and improves image visibility compared to Francisco Sans', Roba Binyahibs', Feiniu Yuan's, Leonardo's, and Yan Zhang's algorithms [S2].

### Geometry Sampling Adjustment
Geometry Sampling Adjustment is a method to adjust the maximum number of geometry samples per streamline based on a geometry sampling factor [S3, unofficial]. It sets the maximum geometry samples $m_{maxGeometrySamples}$ to $m_{maxSamples} / m_{geometrySampling} + 1$, where $m_{maxSamples}$ is the base maximum number of samples per streamline and $m_{geometrySampling}$ is a factor used to reduce the number of geometry samples [S3, unofficial].

## Operating characteristics
*   **Performance:** Provides small performance gains in sparse volumes by reducing the number of texture fetches in empty space [S1, unofficial].
*   **Quality:** May introduce artifacts in some cases, particularly if the doubling rate is too aggressive or the mipmap transition is visible [S1, unofficial].
*   **Complexity:** Low computational overhead; requires only a counter and simple arithmetic operations per step.
*   **Use Case:** Best suited for volumetric effects with large empty regions, such as atmospheric clouds or fog, where density drops to zero over significant distances [S1, unofficial].

## Background
The technique is commonly used in real-time graphics for atmospheric rendering, such as in the implementation of volumetric clouds in game engines like Unity's HDRP, where it is referred to as "Dynamic Sampling Step Adjustment" [S1, unofficial]. It is also related to adaptive sampling strategies in medical image visualization, such as the Dynamic Adaptive Sampling Ray-Casting algorithm proposed by Wang et al. [S2].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] A Novel Ray-Casting Algorithm Using Dynamic Adaptive Sampling, Huadeng Wang, Guang Xu, Xipeng Pan, Xiaonan Luo, WIREL COMMUN MOB COM, Oct 2020, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S3] NightElfik/Vector-field-visualization-using-cuda, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
