# Ambient Occlusion via SDF

Also called: AO Ray Marching, Alex Evans AO, Cheap Ambient Occlusion, Normal-direction AO, Proximity-based Occlusion, Ray Marched Ambient Occlusion, Real Ambient Occlusion, SDF AO

## Overview
Ambient Occlusion via SDF approximates the darkening of crevices and contact points by sampling the signed distance function (SDF) along the surface normal [S1, S3, S4, unofficial]. It detects occlusion by comparing the actual distance to nearby geometry against the expected distance if the surface were isolated; a smaller actual distance indicates an obstacle is closer, reducing the ambient light contribution [S2, S5, unofficial]. Unlike screen-space methods or shadow maps, this technique requires no separate rendering pass or auxiliary buffers, relying solely on the scene's distance function [S6, unofficial]. It is computationally cheaper than full ray-marched shadows but provides a local approximation of global illumination effects [S4, unofficial].

## How it works
**Inputs:**
*   `p`: The 3D position of the surface point being shaded.
*   `n`: The normalized surface normal at `p`.
*   `map(p)`: The scene SDF function returning the distance to the nearest surface.

**Outputs:**
*   `ao`: A scalar value in $[0, 1]$, where 0 is fully occluded (black) and 1 is fully lit.

**Data Structures:**
*   No complex structures. Standard floating-point variables.

**Parameters:**
*   `num_steps`: Number of samples along the normal. Typical values: 5 [S2, S5, unofficial], 20 [S3, unofficial].
*   `step_dist`: Base step size. Typical values: 0.015 [S3, unofficial], 0.08 [S5, unofficial], or derived via linear interpolation [S2, unofficial].
*   `decay`: Exponential decay factor for sample weights. Typical values: 0.6 [S1, unofficial], 0.7 [S2, S5, unofficial].
*   `multiplier`: Scaling factor applied to the accumulated occlusion before clamping. Typical values: 1.4 [S2, unofficial], 2.0 [S5, unofficial].
*   `power`: Exponent applied to the final AO term to amplify contrast. Typical value: 40 [S3, unofficial].

**Algorithm:**
The method casts a ray from `p` along `n` in discrete steps. At each step $i$, it evaluates the SDF at position $p_i = p + n \cdot d_i$. The occlusion is determined by the difference between the step distance $d_i$ and the SDF value $h_i = \text{map}(p_i)$. If $h_i < d_i$, another surface is closer than the empty space expected, indicating occlusion [S3, S5, unofficial].

There are three common variations in implementation found in the sources:

1.  **Linear Step with Exponential Decay (Alex Evans style):** Samples at fixed intervals with decaying weights. This is the most common "cheap" AO [S2, S5, unofficial].
2.  **Squared Difference with Inverse Distance Weighting:** Uses a squared difference term divided by the step index to penalize closer obstacles more heavily while reducing influence of distant samples [S3, unofficial].
3.  **Sum Ratio:** Sums the SDF values and divides by the expected sum if no occlusion existed [S4, unofficial].

**Pseudocode (Linear Step with Exponential Decay):**
This variant is preferred for its stability and widespread use in real-time raymarching [S2, S5, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ambient_occlusion_sdf(p, n, num_steps=5, base_step=0.08, decay=0.7, multiplier=2.0):
    """
    Computes ambient occlusion by sampling the SDF along the normal.
    
    Args:
        p: vec3, surface position.
        n: vec3, surface normal (normalized).
        num_steps: int, number of samples (typically 5).
        base_step: float, base distance increment (typically 0.08).
        decay: float, weight decay per step (typically 0.7).
        multiplier: float, scaling factor for occlusion intensity (typically 2.0).
        
    Returns:
        float: Ambient occlusion factor in [0, 1].
    """
    occ = 0.0
    weight = 1.0
    
    for i in range(1, num_steps + 1):
        # Calculate distance for this step
        dist = base_step * i
        
        # Evaluate SDF at the sample point
        # map() is the scene SDF function
        h = map(p + n * dist)
        
        # Accumulate occlusion: difference between expected empty space (dist)
        # and actual distance to geometry (h).
        # If h < dist, geometry is closer, adding to occlusion.
        occ += (dist - h) * weight
        
        # Decay weight for further steps
        weight *= decay
        
    # Scale and clamp result
    # 1.0 - occ gives the lit portion. 
    # multiplier amplifies the effect.
    ao = 1.0 - (occ * multiplier)
    
    return clamp(ao, 0.0, 1.0)
```

**Pseudocode (Squared Difference Variant):**
Used in some tutorials for a different falloff characteristic [S3, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ambient_occlusion_squared(p, n, step_dist=0.015, step_nbr=20):
    """
    Computes AO using squared difference and inverse step weighting.
    """
    occlusion = 1.0
    current_step = step_nbr
    
    while current_step > 0:
        # Distance from origin
        dist = current_step * step_dist
        
        # Sample SDF
        sample_pos = p + n * dist
        sdf_val = map(sample_pos).d # .d implies the distance component
        
        # Calculate contribution
        # Numerator: squared difference between expected distance and actual SDF
        # Denominator: current step index (reduces influence of far samples)
        contribution = pow(dist - sdf_val, 2) / current_step
        
        occlusion -= contribution
        current_step -= 1
        
    return clamp(occlusion, 0.0, 1.0)
```

**Edge Cases and Pitfalls:**
*   **Distant Artifacts:** If the total travel distance (`num_steps * step_dist`) is too large, distant geometry may contribute to the occlusion term, creating incorrect darkening in open areas [S3, unofficial].
*   **Precision:** In high-precision SDFs, small errors in `map()` can cause flickering. Using `highp` float precision is recommended [S1, unofficial].
*   **Normal Accuracy:** The method assumes `n` is accurate. Incorrect normals (e.g., from low-poly meshes without smoothing) lead to sampling in empty space or into geometry incorrectly, producing noisy AO [S6, unofficial].

## Subtopics

### Quick Ambient Occlusion
A variant that estimates AO based on the number of raymarching steps required to hit a surface during the primary ray march, rather than a separate normal-based pass [S4, unofficial].
*   **Mechanism:** During the standard sphere-tracing loop, if the surface is hit at step index $i$ out of a maximum `_Steps`, the AO coefficient is calculated as $1 - \frac{i}{\text{Steps}-1}$ [S4, unofficial].
*   **Pros:** Zero additional SDF evaluations; leverages existing raymarch data.
*   **Cons:** Tightly coupled with raymarch resolution; less accurate for local crevices compared to normal sampling [S4, unofficial].

### Spherical Ambient Occlusion
Estimates AO by sampling the SDF at multiple points distributed on a sphere around the surface point, providing a more isotropic occlusion estimate than normal-only sampling [S7, unofficial].
*   **Mechanism:** Samples are placed using a golden-ratio-based spherical distribution to ensure uniform density [S7, unofficial].
    *   $N$: Number of samples (`_aoSamples`).
    *   $R$: Kernel radius (`_aoKernelSize`).
    *   For $i$ from 0 to $N-1$:
        *   $p_y = 1 - \frac{i}{N} \cdot 2$
        *   $r = \sqrt{1 - p_y^2}$
        *   $\theta = \phi \cdot i$, where $\phi = \pi(3 - \sqrt{5})$
        *   $p_x = \cos(\theta) \cdot r$
        *   $p_z = \sin(\theta) \cdot r$
        *   Sample position: $\text{hitPos} + (p_x, p_y, p_z) \cdot R$
    *   Accumulate: $\text{ao} += \text{saturate}(\frac{\text{SDF}}{R})$
    *   Final AO: $1 - (1 - \frac{\sum \text{saturate}(\text{SDF}/R)}{N \cdot 0.5})^2$
*   **Parameters:** `_aoSamples` typically 8 (range 0-32); `_aoKernelSize` typically 0.075 (range 0.0-0.25) [S7, unofficial].
*   **Pros:** Better handles occlusion from sides, not just above/below the normal.
*   **Cons:** More expensive ($N$ SDF calls per pixel vs 5-20 for normal AO) [S7, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(k)$, where $k$ is the number of AO samples (typically 5-20). It adds a constant overhead to the shading cost per pixel.
*   **Memory Complexity:** $O(1)$. No buffers or textures are required [S6, unofficial].
*   **Performance:** Significantly faster than screen-space ambient occlusion (SSAO) or ray-marched shadows because it avoids large kernel samples or secondary ray tracing through the entire scene [S4, unofficial].
*   **Strengths:**
    *   Resolution-independent (unlike SSAO) [S4, unofficial].
    *   Works well with procedural SDFs and fractals [S6, unofficial].
    *   Easy to tune via `multiplier` and `decay` parameters [S2, S5, unofficial].
*   **Limitations:**
    *   Approximate: Only considers the local neighborhood along the normal (or sphere) [S3, unofficial].
    *   Can produce "halo" artifacts if the step size is too large or if the SDF has sharp discontinuities [S3, unofficial].
    *   Dependent on normal quality [S6, unofficial].

## Background
The technique is rooted in the early days of real-time raymarching in WebGL and GLSL. Alex Evans popularized the "cheap ambient occlusion" method using normal-direction SDF sampling in his 2012 GDC talk "Real-Time Ray Tracing" and subsequent demos [S4, S5, unofficial]. It became a staple in the demoscene and indie shader communities due to its ability to add depth to simple SDF scenes without complex lighting pipelines [S1, S2, unofficial]. The squared-difference variant is often cited in educational tutorials explaining the geometric intuition of AO via SDFs [S3, unofficial].

## Sources
[S1] MohammadHossinzehi/sdf-raymarcher, https://github.com/MohammadHossinzehi/sdf-raymarcher, fetched 2026-10-26 (unofficial)
[S2] PIGJET/neon-raymarcher, https://github.com/PIGJET/neon-raymarcher, fetched 2026-10-27 (unofficial)
[S3] An Introduction to Raymarching - Maxime Garcia, https://typhomnt.github.io/teaching/ray_tracing/raymarching_intro/, fetched 2026-10-27 (unofficial)
[S4] Volumetric Rendering: Ambient Occlusion, https://www.alanzucconi.com/2016/07/01/ambient-occlusion/, fetched 2026-10-28 (unofficial)
[S5] Ray Marching and Signed Distance Functions — GLSL in the..., https://www.mysimulator.uk/content/articles/ray-marching-sdf.html, fetched 2026-10-29 (unofficial)
[S6] Ray Marching and Signed Distance Functions: Sphere Tracing..., https://www.mysimulator.uk/articles/ray-marching/, fetched 2026-10-29 (unofficial)
[S7] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
