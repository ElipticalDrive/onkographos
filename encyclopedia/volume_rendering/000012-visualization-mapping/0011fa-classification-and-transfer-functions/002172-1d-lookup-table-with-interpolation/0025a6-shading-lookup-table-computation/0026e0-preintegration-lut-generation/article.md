# Preintegration LUT Generation

Also called: 2D Preintegration LUT, Preintegration Table Generation

## Overview
Preintegration LUT Generation creates a two-dimensional lookup table that pre-integrates color and opacity over a path length, reducing the number of samples required during volume rendering [S1, unofficial]. It supports two distinct rendering modes: 'medium' mode, which uses a Beer-Lambert approximation with average color, and 'particle' mode, which uses standard front-to-back blending [S1, unofficial]. This method builds on [Shading Lookup Table Computation](3589-shading-lookup-table-computation.md) by moving the integration of the transfer function into a pre-computed texture lookup.

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   `size`: Integer dimension of the 2D LUT. The LUT stores `size * size` entries [S1, unofficial].
    *   `step_length`: Float representing the total path length integrated over in the LUT generation [S1, unofficial].
    *   `is_medium_table`: Boolean flag. If `true`, uses medium rendering mode; if `false`, uses particle rendering mode [S1, unofficial].
    *   Transfer function curves: `medium_colors`, `particle_colors`, and `opacities` [S1, unofficial].
*   **Outputs:**
    *   A 2D texture/array of size `size * size`, where each entry is a `vec4` containing `(R, G, B, Alpha)` [S1, unofficial].

**Data Structures**
1.  **1D Lookup Tables:**
    *   `tf_colors`: Array of `vec4` with `size` entries.
    *   `tf_opacities`: Array of `float` with `size` entries.
2.  **2D LUT:**
    *   `data`: Array of `vec4` with `size * size` entries, indexed by `y * size + x` [S1, unofficial].

**Steps**

1.  **Initialize 1D Lookup Tables:**
    Allocate `tf_colors` and `tf_opacities` with `size` entries [S1, unofficial].

2.  **Populate 1D Tables:**
    For `i` from `0` to `size - 1`:
    1.  Compute normalized parameter: $t = \frac{i}{\text{size} - 1}$ [S1, unofficial].
    2.  Determine color:
        *   If `is_medium_table` is true: `tf_colors[i] = sample_curve(t, tfs.medium_colors)` [S1, unofficial].
        *   Else: `tf_colors[i] = sample_curve(t, tfs.particle_colors)` [S1, unofficial].
    3.  Determine opacity: `tf_opacities[i] = sample_curve(t, tfs.opacities)` [S1, unofficial].

3.  **Generate 2D LUT:**
    For `y` from `0` to `size - 1`:
    For `x` from `0` to `size - 1`:
    1.  Compute path endpoints:
        *   $s_{\text{front}} = \frac{y}{\text{size} - 1}$ [S1, unofficial].
        *   $s_{\text{back}} = \frac{x}{\text{size} - 1}$ [S1, unofficial].
    2.  Initialize accumulators:
        *   `accum_color` = `vec3(0.0)` [S1, unofficial].
        *   `accum_alpha` = `0.0` [S1, unofficial].
        *   `accum_density` = `0.0` [S1, unofficial].
    3.  **Inner Integration Loop:**
        Set `steps = 32` [S1, unofficial].
        Compute step size: $ds = \frac{\text{step\_length}}{\text{steps}}$ [S1, unofficial].
        For `k` from `0` to `steps - 1`:
        1.  Compute interpolation weight: $w = \frac{k + 0.5}{\text{steps}}$ [S1, unofficial].
        2.  Compute interpolated opacity value: $d = s_{\text{front}} \cdot (1.0 - w) + s_{\text{back}} \cdot w$ [S1, unofficial].
        3.  Compute index: `idx` = $\lfloor d \cdot (\text{size} - 1) \rfloor$ [S1, unofficial].
        4.  Fetch values:
            *   `color` = `tf_colors[idx]` [S1, unofficial].
            *   `opacity` = `tf_opacities[idx]` [S1, unofficial].
        5.  **Accumulate based on mode:**
            *   **If `is_medium_table` (Medium Mode):**
                *   `accum_color += color.rgb * opacity` [S1, unofficial].
                *   `accum_density += opacity` [S1, unofficial].
            *   **Else (Particle Mode):**
                *   Compute extinction: $\text{extinction} = \text{opacity} \cdot 100.0$ [S1, unofficial].
                *   Compute sample alpha: $\alpha_{\text{sample}} = 1 - e^{-\text{extinction} \cdot ds}$ [S1, unofficial].
                *   Compute transmittance: $T = 1 - \text{accum\_alpha}$ [S1, unofficial].
                *   Update color: `accum_color += color.rgb * T * alpha_sample` [S1, unofficial].
                *   Update alpha: $\text{accum\_alpha} += (1 - \text{accum\_alpha}) \cdot \alpha_{\text{sample}}$ [S1, unofficial].
    4.  **Finalize LUT Entry:**
        *   **If `is_medium_table`:**
            *   If `accum_density > 0.0001`: `final_color = accum_color / accum_density` [S1, unofficial].
            *   Else: `final_color = vec3(1.0)` [S1, unofficial].
            *   Compute total extinction: $\text{total\_extinction} = \text{accum\_density} \cdot ds \cdot 100.0$ [S1, unofficial].
            *   Compute final alpha: $\alpha_{\text{final}} = 1 - e^{-\text{total\_extinction}}$ [S1, unofficial].
            *   `data[y * size + x] = { final_color.r, final_color.g, final_color.b, final_alpha }` [S1, unofficial].
        *   **Else (Particle Mode):**
            *   `data[y * size + x] = { accum_color.r, accum_color.g, accum_color.b, accum_alpha }` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_preintegration_lut(size, step_length, is_medium_table, tfs):
    # tfs is an object with attributes: medium_colors, particle_colors, opacities
    # sample_curve(t, curve) returns a vec4 (RGBA) or float depending on curve type
    
    # 1. Initialize 1D LUTs
    tf_colors = [vec4(0.0, 0.0, 0.0, 0.0) for _ in range(size)]
    tf_opacities = [0.0 for _ in range(size)]
    
    # 2. Populate 1D LUTs
    for i in range(size):
        t = i / (size - 1)
        if is_medium_table:
            tf_colors[i] = sample_curve(t, tfs.medium_colors)
        else:
            tf_colors[i] = sample_curve(t, tfs.particle_colors)
        tf_opacities[i] = sample_curve(t, tfs.opacities).r # Assuming opacity is R or separate curve
        
    # 3. Generate 2D LUT
    data = [vec4(0.0, 0.0, 0.0, 0.0) for _ in range(size * size)]
    steps = 32
    ds = step_length / steps
    
    for y in range(size):
        s_front = y / (size - 1)
        for x in range(size):
            s_back = x / (size - 1)
            
            accum_color = vec3(0.0, 0.0, 0.0)
            accum_alpha = 0.0
            accum_density = 0.0
            
            for k in range(steps):
                w = (k + 0.5) / steps
                d = s_front * (1.0 - w) + s_back * w
                idx = int(d * (size - 1))
                
                color = tf_colors[idx]
                opacity = tf_opacities[idx]
                
                if is_medium_table:
                    accum_color += color.rgb * opacity
                    accum_density += opacity
                else:
                    extinction = opacity * 100.0
                    alpha_sample = 1.0 - math.exp(-extinction * ds)
                    transmittance = 1.0 - accum_alpha
                    accum_color += color.rgb * transmittance * alpha_sample
                    accum_alpha += (1.0 - accum_alpha) * alpha_sample
            
            if is_medium_table:
                if accum_density > 0.0001:
                    final_color = accum_color / accum_density
                else:
                    final_color = vec3(1.0, 1.0, 1.0)
                total_extinction = accum_density * ds * 100.0
                final_alpha = 1.0 - math.exp(-total_extinction)
                data[y * size + x] = vec4(final_color.r, final_color.g, final_color.b, final_alpha)
            else:
                data[y * size + x] = vec4(accum_color.r, accum_color.g, accum_color.b, accum_alpha)
                
    return data
```

**Parameters**
*   `size`: Integer. Typical values depend on memory constraints and desired accuracy. Not given in the sources.
*   `step_length`: Float. Represents the physical or normalized path length. Not given in the sources.
*   `is_medium_table`: Boolean. Selects between medium and particle integration modes [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Division by Zero:** In medium mode, if `accum_density` is zero or near-zero, the average color calculation is undefined. The method handles this by defaulting to white (`vec3(1.0)`) if `accum_density <= 0.0001` [S1, unofficial].
*   **Precision:** The use of `float` for accumulators and intermediate calculations may introduce precision errors, particularly in the exponential calculations for alpha [S1, unofficial].
*   **Boundary Conditions:** The indices for `tf_colors` and `tf_opacities` are clamped implicitly by the integer cast of `d * (size - 1)`, which should be within `[0, size-1]` if `d` is in `[0, 1]`. If `d` exceeds 1 due to floating point errors, out-of-bounds access may occur [G].

## Subtopics

### LUT Cache Hashing Fix for In-Place Edits
This variant addresses the issue of stale transmittance and multiscattering LUT caches after in-place medium edits. Previously, the cache hashed the medium's handle, which remains constant during in-place asset edits, causing LUTs to go stale until a periodic rebuild every 240 frames. This resulted in a washed-out sky for up to 4 seconds, as sky-view and aerial LUTs already read the new medium [S2, unofficial]. The fix involves hashing the prepared GPU medium's textures instead of the medium's handle. This ensures that re-preparation replaces the textures, correctly invalidating the cache and eliminating the need for the periodic rebuild [S2, unofficial]. Parameters such as `ATMOSPHERE_SECOND_SUN`, `ATMOSPHERE_SUN_AZ_DEG`, `ATMOSPHERE_SUN_SCALE`, `ATMOSPHERE_AEROSOL`, and `ATMOSPHERE_AEROSOL_EDIT_SECS` are relevant in this atmospheric context [S2, unofficial].

### 1D Pre-integration Table Approximation
This approximation technique reduces the dimensionality of the pre-integration lookup table from two dimensions to one, thereby reducing texture memory size and update time [S3]. It modifies the geometric average of transparency into an arithmetic average [S3]. The method relies on the observation that scalar values $s$ are usually quantized, allowing the integral to be rewritten as an arithmetic mean of transmittance [S3].

### Arithmetic Mean Opacity Approximation
This specific step within the 1D approximation uses the arithmetic mean instead of the geometric mean in the integral computation to improve lookup table generation speed [S4]. This simplifies the calculation of color attenuation, allowing it to be computed segment by segment rather than point by point [S4].

## Operating characteristics
*   **Time Complexity:** The generation time is $O(\text{size}^2 \cdot \text{steps})$, where `steps` is the number of integration steps (typically 32) [S1, unofficial].
*   **Memory Complexity:** The 2D LUT requires $O(\text{size}^2)$ memory, with each entry being 16 bytes (4 floats) [S1, unofficial]. For a size of 4096, this would require 64 MB [S3].
*   **Measured Performance:** Not given in the sources.
*   **Strengths:** Reduces the number of samples needed during rendering by pre-integrating the transfer function [S1, unofficial]. The 1D approximation further reduces memory and update time [S3].
*   **Limitations:** The 2D LUT can consume significant memory for large sizes [S3]. The approximation methods may introduce accuracy errors compared to the full 2D integration [S3, S4].
*   **When to use:** When rendering performance is critical and memory is sufficient for the 2D LUT, or when using the 1D approximation for memory-constrained environments.
*   **When not to use:** When high accuracy is required and the approximation errors are unacceptable, or when the transfer function changes frequently and the LUT generation overhead is prohibitive.

## Background
Preintegration was introduced to improve the performance of volume rendering by moving the integration of the transfer function into a pre-computed lookup table [S3]. The method was proposed in "Efficient Interactive Pre-integrated Volume Rendering" [S3]. The technique has been further developed with approximations such as the 1D pre-integration table and arithmetic mean opacity approximation to reduce memory usage and improve generation speed [S3, S4]. The implementation details provided in the notes are from an unofficial source [S1, unofficial] and a GitHub pull request regarding atmospheric volumetric clouds [S2, unofficial].

## Sources
[S1] GitThirteen/vis-magnus-2018, https://github.com/GitThirteen/vis-magnus-2018, fetched 2026-10-26 (unofficial)
[S2] WIP: Nubis-based volumetric clouds, https://github.com/mate-h/bevy/pull/26, fetched 2026-10-27 (unofficial)
[S3] Efficient Interactive Pre-integrated Volume Rendering, https://doi.org/10.1007/11428862_118, fetched 2026-10-26
[S4] Volume Visualization: A Technical Overview with a Focus on ..., https://pmc.ncbi.nlm.nih.gov/articles/PMC3138940/, fetched 2026-10-27
