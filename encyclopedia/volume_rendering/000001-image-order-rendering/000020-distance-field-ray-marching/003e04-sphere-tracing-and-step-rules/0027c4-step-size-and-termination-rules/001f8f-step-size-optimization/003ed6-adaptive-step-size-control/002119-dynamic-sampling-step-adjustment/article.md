# Dynamic Sampling Step Adjustment

Also called: Adaptive Step Doubling, Empty Space Step Doubling

## Overview
Dynamic Sampling Step Adjustment is a heuristic optimization for distance-field ray marching that accelerates traversal through empty or low-density regions by exponentially increasing the step size. It solves the problem of redundant sampling in sparse volumes, such as atmospheric clouds or distant geometry, where uniform steps waste computational resources [S1, unofficial][S2]. It sets itself apart from gradient-driven methods by relying on occupancy/density checks rather than field derivatives, and from fixed adaptive schemes by using a multiplicative doubling strategy that reacts to consecutive empty samples [S1, unofficial].

## How it works
The method operates on a ray $r(t) = \mathbf{o} + t\mathbf{d}$ traversing a volume or SDF. It maintains a current step size $\Delta t$ and a sampling offset (often mapped to a Mipmap level `mipOffset` in texture-based implementations).

**Inputs:**
- Ray origin $\mathbf{o}$ and direction $\mathbf{d}$.
- Scene function $f(\mathbf{p})$ returning density or signed distance.
- Initial step size $\Delta t_0$.
- Maximum step size $\Delta t_{max}$ (optional, often implicit).
- Empty threshold $\epsilon_{empty}$ (for density) or $\epsilon_{hit}$ (for SDF).

**Outputs:**
- Accumulated distance $t$.
- Final sample position and color/opacity.

**Data Structures:**
- `current_step`: float, initialized to $\Delta t_0$.
- `mip_offset`: integer or float, tracking the resolution level for texture lookups.
- `consecutive_empty`: integer, counter for empty samples.

**Algorithm Steps:**
1. Initialize $t = 0$, `current_step` = $\Delta t_0$, `mip_offset` = 0, `consecutive_empty` = 0.
2. While $t < t_{max}$ and not converged:
    a. Evaluate sample at $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.
    b. Query value $v = f(\mathbf{p})$.
    c. Check if sample is "empty":
       - For SDFs: $v > \epsilon_{hit}$ (or similar threshold indicating no surface nearby).
       - For Volumes: $v < \epsilon_{empty}$ (density below threshold).
    d. If empty:
       i. Increment `consecutive_empty`.
       ii. If `consecutive_empty` > 0 (or a specific threshold, often 1 or 2):
           - Double the step size: `current_step` = `current_step` * 2.
           - Increase `mip_offset` (e.g., `mip_offset` += 1) to sample lower resolution, reducing texture bandwidth [S1, unofficial].
           - Clamp `current_step` to $\Delta t_{max}$ if defined.
       iii. Advance $t$ by `current_step`.
    e. If not empty:
       i. Step back: $t = t - \text{previous\_step}$ (or simply do not advance by the doubled step; the ray must re-evaluate the boundary). *Note: The source [S1, unofficial] states "step back one step". In implementation, this usually means reverting the last advancement or ensuring the next step is small enough to catch the boundary. A robust implementation resets the step size.*
       ii. Reset `current_step` to $\Delta t_0$.
       iii. Reset `mip_offset` to 0.
       iv. Reset `consecutive_empty` to 0.
       v. Advance $t$ by `current_step` (the original small step).
    f. Check termination conditions (hit, escape, max iterations).

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_sampling_step_adjustment(ray_origin, ray_dir, scene_func, t_max, initial_step, empty_threshold):
    t = 0.0
    step = initial_step
    mip_offset = 0
    consecutive_empty = 0
    
    # Store previous step to handle "step back" logic if necessary, 
    # though typically we just reset step size on hit.
    prev_step = initial_step 
    
    while t < t_max:
        # Sample position
        pos = ray_origin + ray_dir * t
        
        # Evaluate scene function (density or SDF)
        val = scene_func(pos, mip_offset)
        
        # Check for empty/transparent region
        is_empty = (val < empty_threshold)
        
        if is_empty:
            consecutive_empty += 1
            
            # Double step size and increase mip level
            # Note: Source [S1, unofficial] implies doubling happens on consecutive empties.
            # We clamp step to prevent overshooting the entire volume.
            step = min(step * 2, t_max - t) 
            
            # Increase mip offset for lower resolution sampling
            mip_offset += 1
            
            # Advance ray
            t += step
        else:
            # Non-empty region detected
            # Step back: The source [S1, unofficial] says "step back one step".
            # This implies we must ensure we don't skip the boundary.
            # A common implementation is to reset step size and re-evaluate 
            # or advance by the original small step.
            
            # Reset parameters
            step = initial_step
            mip_offset = 0
            consecutive_empty = 0
            
            # "Step back" interpretation:
            # If we just jumped over the boundary, we need to go back.
            # However, standard ray marching usually just reduces step size.
            # To strictly follow "step back one step", we subtract the previous large step?
            # No, the loop structure suggests we are at 't'. 
            # If we detected non-empty, we are inside. We should advance by the small step.
            # The "step back" likely refers to the logic of not accepting the large jump 
            # that led to the empty check, or resetting the state.
            # Given the ambiguity, we reset step and advance by initial_step.
            # If the previous jump was huge and landed inside, we might have skipped details.
            # A safer interpretation of "step back one step" in this context is:
            # Revert the last advancement if it was the one that caused the check?
            # Actually, if we are in a loop, 't' is the current position.
            # If we are NOT empty, we are in the volume. We proceed with small steps.
            
            # To implement "step back one step" literally:
            # If the previous iteration was a "large jump" that landed here,
            # we might need to go back to the start of that jump?
            # Let's assume the standard behavior: reset step, advance by small step.
            # The "step back" might be a translation artifact meaning "revert to small step mode".
            
            t += step
            
        # Termination checks would go here
        if is_surface_hit(val):
            break
            
    return t
```

**Parameters:**
- `initial_step`: Typical value depends on volume resolution. For SDFs, often related to the minimum feature size.
- `empty_threshold`: For SDFs, often a small epsilon like $10^{-4}$. For volumes, near 0.
- `mip_offset`: Starts at 0. Increments with empty steps.

**Edge Cases and Pitfalls:**
- **Artifacts:** The optimization provides small performance gains and may cause artifacts in some cases, particularly if the step size doubles too aggressively and skips thin features [S1, unofficial].
- **Oscillation:** If the boundary is noisy, the step size may oscillate between large and small values.
- **Precision:** Large steps may suffer from floating-point precision issues when adding to $t$.

## Subtopics

### [Gradient-driven Sampling](5281-gradient-driven-sampling.md)
Gradient-driven Sampling is a dynamic step size adjustment technique for distance-field ray marching that refines traversal steps according to the magnitude of the scalar field gradient $|\nabla s|$. It solves the problem of balancing rendering accuracy against computational cost in volumes with sharp boundaries, such as CT angiography data, by taking smaller steps in high-gradient regions to capture transitions accurately and larger steps in smooth interiors. It sets itself apart from uniform sampling by adapting to local feature sharpness, achieving 2-3x efficiency gains without aliasing artifacts. Link: [Gradient-driven Sampling](5281-gradient-driven-sampling.md).

### [Dynamic Texture Slice Resampling](6101-dynamic-texture-slice-resampling.md)
Dynamic Texture Slice Resampling is a technique for volume rendering that dynamically updates slicing steps according to viewing directions to maintain a constant step size during the Direct Volume Rendering (DVR) process. It solves the problem of striping artifacts caused by non-uniform sampling steps. The method adapts the sampling density to the viewing geometry to maintain high rendering performance. Link: [Dynamic Texture Slice Resampling](6101-dynamic-texture-slice-resampling.md).

### Dynamic Adaptive Sampling Ray-Casting
Dynamic Adaptive Sampling Ray-Casting is an improved volume rendering algorithm designed to address slow rendering speed and low sampling efficiency in 3D reconstruction of medical images by dynamically adjusting sampling intervals and using a new rendering operator [S2]. It uses the central difference gradient method to obtain the corresponding sampling interval dynamically according to the different sampling points [S2]. It applies a new rendering operator based on the color value and opacity changes before and after the ray enters the volume element, and the resistance luminosity [S2]. On the ISBI2018 dataset, it reduces and avoids artifact phenomena in CT image rendering and improves image visibility compared to Francisco Sans', Roba Binyahibs', Feiniu Yuan's, Leonardo's, and Yan Zhang's algorithms [S2]. Compared with other algorithms, this method achieves faster rendering speed while ensuring the quality of the generated image [S2].

### Geometry Sampling Adjustment
Geometry Sampling Adjustment is a method to adjust the maximum number of geometry samples per streamline based on a geometry sampling factor [S3, unofficial]. It sets `m_maxGeometrySamples` to `m_maxSamples / m_geometrySampling + 1` [S3, unofficial]. `m_maxSamples` is the base maximum number of samples per streamline, and `m_geometrySampling` is a factor used to reduce the number of geometry samples [S3, unofficial].

## Operating characteristics
- **Time Complexity:** Average case is significantly faster than uniform sampling in sparse volumes, as step size grows exponentially. Worst case is similar to uniform sampling if the volume is dense.
- **Memory Complexity:** Low, as it primarily modifies the step size and mip level, requiring no additional large data structures.
- **Performance:** Provides small performance gains [S1, unofficial]. The gains are most noticeable in large empty spaces (e.g., atmospheric clouds) [S1, unofficial].
- **Strengths:** Simple to implement; effective for empty space skipping; reduces texture bandwidth by using higher mip levels [S1, unofficial].
- **Limitations:** May cause artifacts if step size increases too quickly, skipping thin structures [S1, unofficial]. Not suitable for volumes with fine, high-frequency details throughout.
- **When to use:** Large volumes with significant empty space, such as atmospheric rendering or distant SDF scenes.
- **When not to use:** Dense volumes with fine details, or when artifact-free rendering is critical for thin structures.

## Background
The concept of dynamic step adjustment in ray marching is rooted in the need to optimize rendering speed for complex 3D scenes. The specific "Empty Space Step Doubling" technique is documented in the context of real-time volume cloud rendering in games, where it is used to optimize the traversal of cloud layers [S1, unofficial]. The broader concept of Dynamic Adaptive Sampling (DAS) for ray-casting in medical image processing was proposed by Wang et al. in 2020 to address slow rendering speeds in 3D reconstruction [S2]. Geometry sampling adjustments are also used in vector field visualization to control sample density along streamlines [S3, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] A Novel Ray-Casting Algorithm Using Dynamic Adaptive Sampling, https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S3] NightElfik/Vector-field-visualization-using-cuda, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
