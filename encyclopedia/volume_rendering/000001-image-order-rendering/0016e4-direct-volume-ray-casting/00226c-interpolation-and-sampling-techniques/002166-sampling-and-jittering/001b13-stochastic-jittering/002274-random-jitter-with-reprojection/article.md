# Random Jitter with Reprojection

Also called: Frame Accumulation, Temporal Reprojection

## Overview
Random Jitter with Reprojection is a technique to reduce banding artifacts caused by low sample counts in volume rendering by introducing stochastic jitter during ray marching and subsequently reducing the resulting noise via temporal accumulation [S1, unofficial]. It distinguishes itself from pure spatial jittering methods by leveraging temporal coherence, projecting a pseudo-hit position from the current frame into the previous frame's screen space to blend the current result with a history buffer, thereby allowing for lower per-frame sample counts while maintaining visual stability [S1, unofficial].

## How it works
The method modifies the standard marching loop to introduce jittered sampling positions and maintains a specific data structure for reprojection [S1, unofficial].

**Inputs and Outputs:**
- Input: Current view matrix, previous view matrix, current frame's accumulated color, history buffer (previous frame's color), step size, density threshold [S1, unofficial].
- Output: Final accumulated color for the current pixel [S1, unofficial].

**Data Structures:**
- `reprojectionHitPosition`: A `float3` variable storing the 3D world position, updated during the march loop when the accumulated density falls below a threshold, serving as a proxy for the hit position in scenes without explicit geometry [S1, unofficial].

**Parameters:**
- `TRESHOLD`: A density threshold used to determine if the ray is still within the volume (e.g., cloud) for reprojection purposes [S1, unofficial].
- `weight`: A blending factor between the history buffer and the current result, kept relatively high because volumetric elements like clouds do not reproject accurately due to their non-rigid nature [S1, unofficial].

**Steps:**
1. Initialize `reprojectionHitPosition` to a default value (e.g., origin or previous frame's hit) [G].
2. Iterate through the ray march loop:
   a. Calculate a jittered position by adding a random float multiplied by the step size to the current sample position [S1, unofficial].
   b. Update the base `samplePosition` by adding the step size (without jitter) to maintain a non-jittered progression track [S1, unofficial].
   c. Sample the density at the jittered position [S1, unofficial].
   d. Accumulate the density [S1, unofficial].
   e. If the accumulated density is below `TRESHOLD`, update `reprojectionHitPosition` to the current non-jittered `samplePosition` [S1, unofficial].
3. After the march loop completes, project `reprojectionHitPosition` to the previous screen space using the previous view matrix [S1, unofficial].
4. Sample the history buffer at the projected screen position to retrieve the previous frame's result [S1, unofficial].
5. Blend the history result and the current frame's result using the weight factor [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pixel(ray_origin, prev_view, history_buffer, step_size, threshold, max_steps):
    reprojection_hit_position = float3(0.0, 0.0, 0.0)
    sample_position = ray_origin
    accumulated_density = 0.0
    current_result = float3(0.0, 0.0, 0.0)
    weight = 0.8  # [G] High weight for volumetrics

    for step in range(max_steps):
        jitter_position = sample_position + random_float() * step_size
        sample_position += step_size
        sample_density = sample_density_function(jitter_position)  # [G] Renderer-specific density lookup
        accumulated_density += sample_density
        current_result += compute_color(sample_density)  # [G] Renderer-specific color accumulation

        if accumulated_density < threshold:
            reprojection_hit_position = sample_position

    prev_screen_pos = world_to_screen(reprojection_hit_position, prev_view)
    history = history_buffer.sample(prev_screen_pos)
    result = lerp(history, current_result, weight)
    return result
```

**Edge Cases and Pitfalls:**
- Clouds and Volumetrics: Since clouds lack a sharp geometric boundary, the hit position is an approximation [S1, unofficial]. Inaccurate reprojection leads to ghosting or blurring [S1, unofficial]. The blending weight must be tuned to balance noise reduction against motion artifacts [S1, unofficial].
- Threshold Selection: If `TRESHOLD` is too high, the reprojection position may be set too early (shallow in the cloud), leading to misalignment [S1, unofficial].
- Precision: Floating-point precision in world-to-screen projection can cause jitter in the history buffer sampling if the camera moves significantly between frames [G].

## Operating characteristics
- Time Complexity: $O(N)$ per pixel, where $N$ is the number of march steps [G]. The reprojection step adds a constant overhead for matrix multiplication and buffer sampling [G].
- Memory Complexity: $O(W \times H)$ for the history buffer, where $W$ and $H$ are the screen dimensions [G].
- Performance: Allows for a significant reduction in per-frame sample counts compared to non-jittered methods, reducing computational cost while maintaining visual quality through temporal accumulation [S1, unofficial].
- Strengths: Effective at reducing banding artifacts from low sample counts; leverages temporal coherence to smooth noise [S1, unofficial].
- Limitations: Introduces temporal latency and potential ghosting artifacts, especially with fast camera motion or dynamic volumetric changes [S1, unofficial]. Requires a history buffer and reprojection logic [S1, unofficial].
- When to use: When rendering static or slowly moving volumetric scenes (e.g., clouds) where temporal coherence is high and banding from low sample counts is a concern [S1, unofficial].
- When not to use: In scenes with rapid camera motion or highly dynamic volumetric changes where ghosting artifacts would be distracting [S1, unofficial].

## Background
This technique is described in the context of volumetric cloud rendering, specifically as an adaptive sampling technique to reduce the cost of high-quality cloud rendering by lowering sample counts and using temporal accumulation to mitigate noise [S1, unofficial].

## Sources
[S1] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
