# Dynamic Sampling Step Adjustment

Also called: Adaptive Step Doubling, Empty Space Step Doubling

## Overview
Dynamic Sampling Step Adjustment is a heuristic optimization for distance-field ray marching that accelerates traversal through empty or low-density regions by exponentially increasing the step size [S1, unofficial]. It solves the problem of redundant sampling in large voids, which is a common bottleneck in volume rendering and sphere tracing [G].

## How it works
**Inputs and outputs:**
- Input: Ray origin $\mathbf{o}$, normalized direction $\mathbf{d}$ [G], maximum ray distance $t_{\max}$, initial step size $\Delta t_{\text{init}}$, empty density threshold $\epsilon$, and a field sampling function $S(\mathbf{p}, m)$ that returns the density at position $\mathbf{p}$ and mip-map level $m$ [G].
- Output: Final ray parameter $t$, final position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$, final step size $\Delta t$, and final mip-map offset $m$.

**Data structures:**
- $\Delta t$: Floating-point scalar representing the current integration step length in world units [G].
- $m$: Integer index representing the current mip-map level of the sampled texture or SDF [G].
- $\Delta t_{\text{orig}}$: Floating-point scalar storing the base step size when entering a non-empty region [G].
- $m_{\text{orig}}$: Integer storing the base mip-map offset [G].

**Algorithm steps:**
1. Initialize $t = 0$, $\Delta t = \Delta t_{\text{init}}$, $m = 0$, $\Delta t_{\text{orig}} = \Delta t_{\text{init}}$, and $m_{\text{orig}} = 0$.
2. While $t < t_{\max}$:
   a. Compute current position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.
   b. Query density $v = S(\mathbf{p}, m)$.
   c. If $v < \epsilon$ (region is empty):
      - Double the step size: $\Delta t \leftarrow \Delta t \times 2$.
      - Increase the mip-map offset: $m \leftarrow m + 1$.
      - Advance the ray: $t \leftarrow t + \Delta t$.
   d. Else (region is non-empty):
      - Step back one iteration: $t \leftarrow t - \Delta t$.
      - Restore base parameters: $\Delta t \leftarrow \Delta t_{\text{orig}}$, $m \leftarrow m_{\text{orig}}$.
      - Advance the ray with the restored step: $t \leftarrow t + \Delta t$.
3. Return $t$, $\mathbf{p}$, $\Delta t$, and $m$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_sampling_step_adjustment(ray_origin, ray_direction, max_dist, initial_step, empty_threshold, sample_field):
    """
    Ray marching with dynamic step adjustment based on density emptiness.
    sample_field(pos, mip_offset) -> float: Returns density at pos and mip level.
    """
    t = 0.0
    step_size = initial_step
    mip_offset = 0
    original_step = initial_step
    original_mip_offset = 0

    while t < max_dist:
        pos = ray_origin + ray_direction * t
        density = sample_field(pos, mip_offset)

        if density < empty_threshold:
            step_size *= 2.0
            mip_offset += 1
            t += step_size
        else:
            t -= step_size
            step_size = original_step
            mip_offset = original_mip_offset
            t += step_size

    return t, pos
```

**Parameters:**
- `initial_step`: Base step size in world units when entering a non-empty region. Not given in the sources.
- `empty_threshold`: Density value below which a region is considered empty, typically $0$ or a small epsilon [S1, unofficial].
- `max_dist`: Maximum ray length for termination [G].

**Edge cases and pitfalls:**
- **Overshooting:** Doubling the step size can cause the ray to jump over thin geometry or high-frequency details if the initial step is too large or the doubling factor is aggressive [S1, unofficial].
- **Artifacts:** The optimization provides small performance gains and may cause visible artifacts in some scenes [S1, unofficial].
- **Mip-map transitions:** Rapidly increasing the mip-map offset can introduce aliasing or detail loss when transitioning between empty and dense regions [G].
- **Termination clamping:** In practice, $\Delta t$ should be clamped to $t_{\max} - t$ to prevent overshooting the ray endpoint [G].

## Subtopics

### [Gradient-driven Sampling](5281-gradient-driven-sampling.md)
Gradient-driven sampling is a dynamic step size adjustment method for distance-field ray marching that varies the integration step length based on the local magnitude of the scalar field gradient. It solves the problem of balancing rendering fidelity and performance in volumes with sharp boundaries (e.g., CT angiography) by taking smaller steps near high-gradient regions to capture transitions accurately and larger steps in smooth interiors. It sets itself apart from uniform sampling by adapting to local feature sharpness and from hierarchical sampling by using a continuous gradient magnitude threshold rather than discrete variance-based subdivision.

### [Dynamic Texture Slice Resampling](6101-dynamic-texture-slice-resampling.md)
Dynamic Texture Slice Resampling is a technique for volume rendering that dynamically updates slicing steps according to viewing directions to maintain a constant step size during the Direct Volume Rendering (DVR) process. It solves the problem of striping artifacts caused by non-uniform sampling steps, which occur when fixed step sizes result in varying spatial resolution across the volume due to perspective projection. Unlike methods that rely on post-processing upsampling or fixed-step approximations, this approach maintains high rendering performance by adapting the sampling density to the viewing geometry in real-time.

### Dynamic Adaptive Sampling Ray-Casting
An improved volume rendering algorithm designed to address slow rendering speed and low sampling efficiency in 3D reconstruction of medical images by dynamically adjusting sampling intervals and using a new rendering operator [S2]. It uses the central difference gradient method to obtain the corresponding sampling interval dynamically according to the different sampling points [S2]. It applies a new rendering operator based on the color value and opacity changes before and after the ray enters the volume element, and the resistance luminosity [S2]. On the ISBI2018 dataset, the ray-casting DAS algorithm reduces and avoids artifact phenomena in CT image rendering and improves image visibility compared to Francisco Sans' algorithm, Roba Binyahibs' algorithm, Feiniu Yuan's algorithm, Leonardo's algorithm, and Yan Zhang's algorithm [S2]. Compared with other algorithms, this method achieves faster rendering speed while ensuring the quality of the generated image [S2].

### Geometry Sampling Adjustment
A method to adjust the maximum number of geometry samples per streamline based on a geometry sampling factor [S3, unofficial]. It sets `m_maxGeometrySamples` to `m_maxSamples / m_geometrySampling + 1` [S3, unofficial]. `m_maxSamples` is the base maximum number of samples per streamline, and `m_geometrySampling` is a factor used to reduce the number of geometry samples [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Reduces the number of texture/SDF lookups in empty space by a factor proportional to the size of the empty region [G]. In dense regions, it reverts to the base step size, so complexity matches uniform sampling [G].
- **Memory Complexity:** Minimal overhead; requires storing current step size, mip offset, and their original values [G].
- **Performance:** Provides small performance gains [S1, unofficial].
- **Limitations:** May cause artifacts [S1, unofficial]. Not suitable for scenes with thin geometry that might be skipped by large steps [S1, unofficial].
- **When to use:** Large volumes with significant empty space, such as atmospheric clouds or outdoor scenes [S1, unofficial].
- **When not to use:** Scenes with high-frequency details or thin structures where step doubling might overshoot [S1, unofficial].

## Background
The concept of dynamic step adjustment in the context of empty space doubling is described in the context of real-time volume cloud rendering [S1, unofficial]. The broader concept of dynamic adaptive sampling for ray casting in medical imaging was proposed by Wang et al. in 2020 [S2]. Geometry sampling adjustment is found in vector field visualization implementations [S3, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] A Novel Ray-Casting Algorithm Using Dynamic Adaptive Sampling, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S3] NightElfik/Vector-field-visualization-using-cuda, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
