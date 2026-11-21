# Dynamic Sampling Step Adjustment

Also called: Adaptive Step Doubling, Empty Space Step Doubling

## Overview
Dynamic Sampling Step Adjustment is a ray marching optimization technique that accelerates traversal through empty or homogeneous regions of a volume by exponentially increasing the step size when no density is detected [S1, unofficial]. It solves the problem of redundant sampling in sparse areas, such as the sky in atmospheric rendering, by skipping large distances until a feature is encountered [S1, unofficial]. It distinguishes itself from sibling methods like [Adaptive Step Size Ray Marching](2312-adaptive-step-size-ray-marching.md) by using a binary "empty/non-empty" heuristic rather than continuous gradient or variance metrics to drive step changes, and from [Variance-based Step Size Adaptation](13161-variance-based-step-size-adaptation.md) by relying on mip-level offsets rather than statistical error bounds [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, normalized direction $\mathbf{d}$, base step size $\Delta t_{base}$, initial mip level $m_{base}$, and a scene function $V(\mathbf{p}, m)$ that returns density at position $\mathbf{p}$ and mip level $m$.
*   **Output:** Updated ray parameter $t$, final position, and accumulated color/transmittance.

**Data Structures:**
*   `current_step`: Float, current step size $\Delta t$.
*   `current_mip`: Integer, current texture mip level $m$.
*   `consecutive_empty`: Integer, counter for consecutive empty samples.
*   `threshold`: Integer, number of consecutive empty samples required to trigger doubling (typically 2-4, not explicitly given in sources, but implied by "multiple consecutive" [S1, unofficial]).

**Numbered Steps:**
1.  Initialize $t = 0$, $\Delta t = \Delta t_{base}$, $m = m_{base}$, and `consecutive_empty` = 0.
2.  Evaluate density $d = V(\mathbf{o} + t\mathbf{d}, m)$.
3.  Check if $d$ is effectively zero (empty).
    *   If empty:
        *   Increment `consecutive_empty`.
        *   If `consecutive_empty` $\ge$ threshold:
            *   Double the step size: $\Delta t \leftarrow 2 \cdot \Delta t$.
            *   Increase mip level: $m \leftarrow m + 1$ (if $m < m_{max}$).
            *   Reset `consecutive_empty` to 0.
        *   Advance ray: $t \leftarrow t + \Delta t$.
    *   If non-empty:
        *   Step back one step: $t \leftarrow t - \Delta t$.
        *   Restore step size: $\Delta t \leftarrow \Delta t_{base}$.
        *   Restore mip level: $m \leftarrow m_{base}$.
        *   Reset `consecutive_empty` to 0.
        *   Advance ray: $t \leftarrow t + \Delta t$.
4.  Repeat until termination conditions (hit, escape, or max steps) are met.

**Formulas:**
*   Step update: $\Delta t_{new} = 2 \cdot \Delta t_{old}$
*   Mip update: $m_{new} = m_{old} + 1$
*   Position update: $\mathbf{p}_{new} = \mathbf{o} + (t + \Delta t)\mathbf{d}$

**Parameters:**
*   $\Delta t_{base}$: Base step size, typically determined by voxel size or desired resolution [S1, unofficial].
*   $m_{base}$: Initial mip level, usually 0 or the highest resolution level [S1, unofficial].
*   Threshold: Number of consecutive empty samples, not explicitly given in sources [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Artifacts:** The method may cause visual artifacts in some cases, particularly if the doubling skips over thin features or if the mip level change introduces texture aliasing [S1, unofficial].
*   **Performance Gains:** The performance improvement is often small, as the overhead of checking density and managing state may outweigh the savings from skipping samples in moderately dense volumes [S1, unofficial].
*   **Boundary Precision:** Stepping back one step when density is found ensures the feature is not missed, but requires careful handling of floating-point precision to avoid infinite loops or jittering near boundaries [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_sampling_step_adjustment(ray_origin, ray_direction, scene_func, max_dist, max_steps, epsilon, base_step, base_mip, empty_threshold):
    t = 0.0
    step = base_step
    mip = base_mip
    consecutive_empty = 0
    
    for i in range(max_steps):
        if t > max_dist:
            return False, t
        
        p = ray_origin + ray_direction * t
        density = scene_func(p, mip)
        
        # Check if empty (density below epsilon)
        if density < epsilon:
            consecutive_empty += 1
            if consecutive_empty >= empty_threshold:
                # Double step size and increase mip level
                step *= 2.0
                mip += 1
                consecutive_empty = 0
            t += step
        else:
            # Density found: step back and reset
            t -= step
            step = base_step
            mip = base_mip
            consecutive_empty = 0
            t += step
            
    return False, t
```

## Subtopics

### [Gradient-driven Sampling](5281-gradient-driven-sampling.md)
Gradient-driven sampling is a ray marching technique that dynamically adjusts step sizes based on the magnitude of the scalar field gradient, refining traversal in regions with high feature sharpness while coarsening it in smooth interiors. It distinguishes itself from uniform sampling by adapting to local geometry to preserve boundary details without aliasing, and from hierarchical sampling by using gradient magnitude rather than variance or error tolerance to drive step selection.

### [Dynamic Texture Slice Resampling](6101-dynamic-texture-slice-resampling.md)
Dynamic Texture Slice Resampling is a technique for volume rendering that dynamically updates slicing steps according to viewing directions to maintain a constant step size during the Direct Volume Rendering (DVR) process. It solves the problem of striping artifacts caused by non-uniform sampling steps. The method adapts the sampling density to the viewing geometry to maintain high rendering performance.

### [Dynamic Adaptive Sampling Ray-Casting](7687-dynamic-adaptive-sampling-ray-casting.md)
Dynamic Adaptive Sampling Ray-Casting (DAS) is a volume rendering method that accelerates 3D reconstruction by dynamically adjusting sampling intervals based on local voxel gradients and employing a specialized light-blocking composition operator. It addresses the slow rendering speed and low sampling efficiency of standard ray-casting in medical image processing by reducing redundant samples in homogeneous regions while maintaining high precision near isosurfaces. Unlike sibling methods that may ignore voxel correlation or use fixed intervals, DAS explicitly links sampling density to the full increment of voxel value differences and gradient magnitude.

### Geometry Sampling Adjustment
Geometry Sampling Adjustment is a method to adjust the maximum number of geometry samples per streamline based on a geometry sampling factor [S2, unofficial]. It sets `m_maxGeometrySamples` to `m_maxSamples / m_geometrySampling + 1`, where `m_maxSamples` is the base maximum number of samples per streamline and `m_geometrySampling` is a factor used to reduce the number of geometry samples [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Reduces the number of samples in empty regions by a factor of up to $2^k$ where $k$ is the number of doublings, but adds overhead for state management and mip-level switching [S1, unofficial].
*   **Memory Complexity:** Requires storage for the current step size, mip level, and empty counter, which is minimal [S1, unofficial].
*   **Measured Performance:** Performance gains are described as "small" and may be negligible in scenes with significant density [S1, unofficial].
*   **Strengths:** Simple to implement, effective for large empty volumes like sky or space [S1, unofficial].
*   **Limitations:** Can introduce artifacts, especially with mip-level changes; less effective in moderately dense volumes [S1, unofficial].
*   **When to Use:** Best for scenes with large empty regions, such as atmospheric rendering or space simulations [S1, unofficial].
*   **When Not to Use:** Avoid in scenes with thin, high-frequency features or where artifact sensitivity is high [S1, unofficial].

## Background
Dynamic Sampling Step Adjustment originated in the context of real-time volume rendering, particularly for atmospheric effects like clouds in game engines [S1, unofficial]. It was discussed in the context of implementing volume clouds in Unity, leveraging techniques similar to those used in HDRP and other high-end renderers [S1, unofficial]. The method is part of a broader set of optimizations for real-time rendering, including blue noise dithering and temporal upsampling [S1, unofficial].

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S2] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
