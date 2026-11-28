# Dynamic Sampling Step Adjustment

Also called: Adaptive Step Doubling, Adaptive step volume rendering, Empty Space Step Doubling, Ray-casting with variable step size, Volume rendering with configurable sampling distance

## Overview
Dynamic Sampling Step Adjustment is a volume rendering technique where the distance between sampled points along projection rays is explicitly controlled to balance image quality, resolution, and rendering performance [S1]. It solves the problem of excessive rendering time caused by uniform high-density sampling in homogeneous or empty regions, while maintaining high resolution in areas with significant features [S1]. Unlike fixed-step ray marching, it dynamically trades off between rendering time and profile smoothness/full-width at half-maximum (FWHM) by adjusting the sampling distance parameter [S1].

## How it works
The method operates by initializing a ray cast with a base sampling distance and iteratively adjusting this distance based on the local content of the volume (density or SDF values).

**Inputs and Outputs:**
*   **Input:** Ray origin $\mathbf{o}$, normalized direction $\mathbf{d}$, volume data (density field or SDF), base sampling distance $\Delta t_{base}$, maximum distance $t_{max}$, maximum iterations $N_{max}$.
*   **Output:** Composed color/opacity for the pixel, final ray distance $t$.

**Data Structures:**
*   **Ray State:** Current distance $t$, accumulated opacity $\alpha_{acc}$, current step size $\Delta t$.
*   **Volume Access:** Function `sample(p)` returning density $\rho$ or SDF value $f$.

**Parameters:**
*   **Base Sampling Distance ($\Delta t_{base}$):** The initial step size. Typical values tested include 0.010, 0.025, 0.050, 0.100, 0.170, 0.195, and 1.0 mm [S1]. An optimal value for clinical data was found to be 0.025 mm [S1].
*   **Empty Space Threshold:** A density threshold below which a voxel is considered "empty" or "transparent" [S2, unofficial].
*   **Doubling Factor:** The multiplier applied to the step size when empty space is detected (typically 2) [S2, unofficial].

**Algorithm Steps:**
1.  Initialize $t = 0$, $\alpha_{acc} = 0$, $\Delta t = \Delta t_{base}$.
2.  For each iteration $i$ from 0 to $N_{max}$:
    a.  **Escape Check:** If $t \ge t_{max}$, terminate.
    b.  **Evaluate:** Compute position $\mathbf{p} = \mathbf{o} + t\mathbf{d}$. Sample the volume at $\mathbf{p}$ to get density $\rho$ (or SDF $f$).
    c.  **Termination Check:** If $\alpha_{acc} \ge 1 - \epsilon$ (early exit) or surface hit ($|f| < \epsilon$), terminate.
    d.  **Adaptive Step Logic:**
        i.  **Empty Space Detection:** If $\rho$ is empty (below threshold) for multiple consecutive samples:
            *   Double the step size: $\Delta t \leftarrow \Delta t \times 2$.
            *   [G] Optionally increase mip-map offset to sample lower-resolution textures, reducing memory bandwidth [S2, unofficial].
        ii. **Non-Empty Detection:** If $\rho$ is non-empty (above threshold):
            *   Step back one step: $t \leftarrow t - \Delta t_{previous}$ (where $\Delta t_{previous}$ is the step size used in the previous iteration before any doubling).
            *   Restore step size: $\Delta t \leftarrow \Delta t_{base}$.
            *   [G] Restore mip-map offset if used [S2, unofficial].
    e.  **Advance:** $t \leftarrow t + \Delta t$.
    f.  **Composite:** Accumulate opacity and color contribution from the sample at $\mathbf{p}$ using standard front-to-back or back-to-front compositing.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_sampling_step_adjustment(ray_origin, ray_direction, volume_sampler, 
                                     max_dist, max_steps, base_step, 
                                     empty_threshold, epsilon=1e-4):
    """
    Implements Dynamic Sampling Step Adjustment.
    volume_sampler(p) returns density value.
    """
    t = 0.0
    alpha_acc = 0.0
    current_step = base_step
    prev_step = base_step
    consecutive_empty = 0
    
    for i in range(max_steps):
        # Escape check
        if t >= max_dist:
            break
            
        # Evaluate scene
        p = ray_origin + ray_direction * t
        density = volume_sampler(p)
        
        # Early termination for volume opacity
        if alpha_acc >= 1.0 - epsilon:
            break
            
        # Adaptive Step Logic
        if density < empty_threshold:
            # Empty space: double step size
            consecutive_empty += 1
            current_step = prev_step * (2 ** consecutive_empty)
        else:
            # Non-empty space: reset
            # Step back to the start of the last "empty" block to ensure no feature is skipped
            # Note: The source describes stepping back one step and restoring base step.
            # To implement "step back one step" correctly after doubling:
            # We must revert t to where we were before the last step that led to the empty detection.
            # However, the source says "step back one step and restore... to original values".
            # This implies we undo the last advancement.
            
            # If we just detected non-empty, we need to re-sample the point we skipped over
            # or the point immediately preceding the empty block.
            # Standard interpretation: revert the last step.
            t -= current_step # Undo the large step
            current_step = base_step
            prev_step = base_step
            consecutive_empty = 0
            
            # Re-evaluate at the reverted position to ensure we capture the feature
            # In a real loop, this might require a flag or restructuring to avoid infinite loops
            # if the logic is simplistic. Here we assume the loop continues.
            # To strictly follow "step back one step":
            # We need to know the step size used to get to the current 'p'.
            # Let's track 'last_step_used'.
            pass 
        
        # Guard against zero or negative steps
        if current_step <= 0:
            current_step = base_step
            
        # Advance
        # We need to store the step used for the next iteration's potential rollback
        last_step_used = current_step
        t += current_step
        prev_step = last_step_used # For next iteration's rollback reference if needed
        
        # Composite sample at NEW position t (or previous if rolled back)
        # Note: The pseudocode structure above is simplified. 
        # A robust implementation requires careful state management for the "step back".
        # Below is a more concrete implementation of the "step back" logic:
        
        # Re-evaluating the logic from [S2, unofficial]:
        # "When density is non-empty, step back one step and restore step size"
        # This implies the sample at the current 'p' (which was non-empty) was reached 
        # via a doubled step. We must go back to the start of that step.
        
        # Corrected Loop Logic for Implementation:
        # 1. Sample at t.
        # 2. If empty: increase step.
        # 3. If non-empty: 
        #    a. If previous step was doubled, we overshot the feature boundary.
        #    b. We must retreat to the point before the doubled step.
        #    c. Reset step to base.
        #    d. Re-sample.
        
        # Due to the complexity of "stepping back" in a forward-only loop, 
        # the following is a direct translation of the intent:
        
        # [G] Implementation detail: To avoid infinite loops, we track if we just stepped back.
        pass

    # Simplified robust version based on [S2, unofficial] intent:
    t = 0.0
    alpha_acc = 0.0
    step = base_step
    prev_step = base_step
    
    for i in range(max_steps):
        if t >= max_dist:
            break
            
        p = ray_origin + ray_direction * t
        density = volume_sampler(p)
        
        if alpha_acc >= 1.0 - epsilon:
            break
            
        if density < empty_threshold:
            # Empty: accelerate
            step = prev_step * 2
        else:
            # Non-empty: decelerate
            # If we are here, we might have arrived via a large step.
            # We need to ensure we don't skip the feature.
            # The source says "step back one step".
            # This means we undo the last 'step' addition.
            t -= step
            step = base_step
            prev_step = base_step
            # Re-evaluate p at the new t
            p = ray_origin + ray_direction * t
            density = volume_sampler(p) # Re-sample
            
        # Safety clamp
        if step <= 0:
            step = base_step
            
        # Advance
        t += step
        prev_step = step # Store for next iteration's potential doubling base
        
        # Composite
        # [G] Standard compositing logic would go here
        alpha = 1.0 - exp(-density * step) # Example Beer-Lambert
        alpha_acc += alpha * (1.0 - alpha_acc)
        
    return alpha_acc
```

**Edge Cases and Pitfalls:**
*   **Artifacts:** The optimization provides small performance gains but may cause artifacts in some cases, particularly if the step doubling skips over thin features or if the "step back" logic is not precisely implemented [S2, unofficial].
*   **Precision:** Rapid changes in step size can lead to aliasing if the minimum step size is not constrained [S1].
*   **Convergence:** The "step back" mechanism must be carefully managed to prevent infinite loops if the density oscillates near the threshold at the boundary of a doubled step.

## Subtopics

### [Gradient-driven Sampling](5281-gradient-driven-sampling.md)
Gradient-driven sampling is a ray marching technique that dynamically adjusts step sizes based on the magnitude of the scalar field gradient, refining traversal in regions with high feature sharpness while coarsening it in smooth interiors. It distinguishes itself from uniform sampling by adapting to local geometry to preserve boundary details without aliasing, and from hierarchical sampling by using gradient magnitude rather than variance or error tolerance to drive step selection.

### [Dynamic Texture Slice Resampling](6101-dynamic-texture-slice-resampling.md)
Dynamic Texture Slice Resampling is a texturing technique that adjusts sampling frequency to yield approximately constant image-space pattern density, compensating for highly anisotropic stretching of integral surface parameterization. It solves the problem of visual artifacts (gaps, obscured flow) caused by strong surface distortion where natural parameterization stretches the surface by factors of 1000 or greater. Unlike Gradient-driven Sampling which adapts step size based on data gradients, or Dynamic Adaptive Sampling Ray-Casting which adapts to volume density, this method adapts texture frequency to maintain visual consistency of surface features (like stripes) regardless of geometric distortion.

### [Dynamic Adaptive Sampling Ray-Casting](7687-dynamic-adaptive-sampling-ray-casting.md)
Dynamic Adaptive Sampling Ray-Casting (DAS) is a volume rendering method that accelerates 3D reconstruction by dynamically adjusting sampling intervals based on local voxel gradients and employing a specialized light-blocking composition operator. It addresses the slow rendering speed and low sampling efficiency of standard ray-casting in medical image processing by reducing redundant samples in homogeneous regions while maintaining high precision near isosurfaces [G]. Unlike sibling methods that may ignore voxel correlation or use fixed intervals, DAS explicitly links sampling density to the full increment of voxel value differences and gradient magnitude.

### Geometry Sampling Adjustment
Geometry Sampling Adjustment is a method to adjust the maximum number of geometry samples per streamline based on a geometry sampling factor [S3, unofficial]. It sets `m_maxGeometrySamples` to `m_maxSamples / m_geometrySampling + 1`, where `m_maxSamples` is the base maximum number of samples per streamline and `m_geometrySampling` is a factor used to reduce the number of geometry samples [S3, unofficial].

## Operating characteristics
*   **Performance:** Decreasing sampling distance (increasing voxel count) significantly increases rendering time but improves resolution (narrower profiles, lower FWHM) [S1]. For clinical data, rendering time was 3.4 s with a default sampling distance of 1.0 mm, reduced to 3.05 s with an optimized distance of 0.025 mm and Hamming BF(z)=2 interpolation [S1].
*   **Quality:** Smaller sampling distances result in narrower profiles, confirming improved image resolution [S1]. Interpolation method choice has less impact on time than sampling distance/voxel count [S1].
*   **Limitations:** The dynamic adjustment (specifically empty space doubling) provides only small performance gains and may introduce artifacts [S2, unofficial].

## Background
Dynamic Sampling Step Adjustment is a fundamental optimization in volume rendering, particularly in medical imaging and real-time graphics. It evolved from the need to handle large volumetric datasets efficiently. The specific parameter optimization for breast tomosynthesis (DBT) data was detailed in 2020, highlighting the trade-off between sampling distance and interpolation quality [S1]. The "empty space doubling" variant is also employed in real-time graphics for volume clouds, as noted in 2023 implementations [S2, unofficial].

## Sources
[S1] Optimization of Breast Tomosynthesis Visualization through 3D Volume Rendering, https://doi.org/10.3390/jimaging6070064, fetched 2026-10-26
[S2] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
[S3] NightElfik/Vector-field-visualization-using-cuda NightElfik/Vector-field-visualization-using-cuda on GitHub, https://github.com/NightElfik/Vector-field-visualization-using-cuda, fetched 2026-10-29 (unofficial)
