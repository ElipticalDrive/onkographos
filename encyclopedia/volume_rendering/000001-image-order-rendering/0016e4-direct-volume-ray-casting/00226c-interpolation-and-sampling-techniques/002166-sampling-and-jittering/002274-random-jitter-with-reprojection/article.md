# Random Jitter with Reprojection

Also called: Frame Accumulation, Temporal Reprojection

## Overview
Random Jitter with Reprojection is a technique to reduce banding artifacts caused by low sample counts in volume rendering by introducing stochastic jitter during ray marching and subsequently reducing the resulting noise via temporal accumulation [S1, unofficial]. It distinguishes itself from pure spatial jittering methods by leveraging temporal coherence, projecting a pseudo-hit position from the current frame into the previous frame's screen space to blend the current result with a history buffer, thereby allowing for lower per-frame sample counts while maintaining visual stability [S1, unofficial].

## How it works
The method operates within a direct volume ray casting framework. It modifies the standard marching loop to introduce jittered sampling positions and maintains a specific data structure for reprojection.

**Inputs and Outputs:**
- Input: Current view matrix, previous view matrix, current frame's accumulated color, history buffer (previous frame's color), step size, density threshold.
- Output: Final accumulated color for the current pixel.

**Data Structures:**
- `reprojectionHitPosition`: A `float3` variable storing the 3D world position. This position is updated during the march loop when the accumulated density is below a specific threshold, serving as a proxy for the "hit" position in scenes without explicit geometry [S1, unofficial].

**Parameters:**
- `TRESHOLD`: A density threshold used to determine if the ray is still within the volume (e.g., cloud) for reprojection purposes. If the accumulated density falls below this value, the current sample position is recorded as the reprojection hit [S1, unofficial].
- `weight`: A blending factor between the history buffer and the current result. This weight is typically kept relatively high because volumetric elements like clouds do not reproject accurately due to their non-rigid, volumetric nature [S1, unofficial].

**Steps:**
1. Initialize `reprojectionHitPosition` to a default value (e.g., origin or previous frame's hit) [G].
2. Iterate through the ray march loop:
   a. Calculate a jittered position by adding a random float multiplied by the step size to the current sample position.
   b. Update the base `samplePosition` by adding the step size (without jitter) to maintain a non-jittered progression track.
   c. Sample the density at the jittered position.
   d. Accumulate the density.
   e. If the accumulated density is below `TRESHOLD`, update `reprojectionHitPosition` to the current non-jittered `samplePosition`.
3. After the march loop completes, project `reprojectionHitPosition` to the previous screen space using the previous view matrix.
4. Sample the history buffer at the projected screen position to retrieve the previous frame's result.
5. Blend the history result and the current frame's result using the weight factor.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_pixel(current_view, prev_view, history_buffer, step_size, threshold):
    # Initialization [G]
    reprojection_hit_position = float3(0, 0, 0)
    sample_position = ray_origin
    accumulated_density = 0.0
    current_result = float3(0, 0, 0)
    
    # March Loop
    while sample_position is within bounds:
        # Jittered sampling
        jitter_position = sample_position + random_float() * step_size
        
        # Update base position without jitter
        sample_position += step_size
        
        # Sample density at jittered position
        sample_density = sample_density_function(jitter_position)
        
        # Accumulate density
        accumulated_density += sample_density
        
        # Update reprojection hit if inside volume threshold
        if accumulated_density < threshold:
            reprojection_hit_position = sample_position
            
        # Accumulate color contribution [G]
        current_result += compute_color(sample_density)
        
        # Break if outside volume or max steps reached [G]

    # Reprojection
    # Project 3D hit position to previous screen space
    prev_screen_pos = world_to_screen(reprojection_hit_position, prev_view)
    
    # Sample history buffer
    history = history_buffer.sample(prev_screen_pos)
    
    # Blend with current result
    # Weight must be relatively high because clouds do not reproject accurately
    result = lerp(history, current_result, weight)
    
    return result
```

**Edge Cases and Pitfalls:**
- **Clouds and Volumetrics:** Since clouds lack a sharp geometric boundary, the "hit" position is an approximation. Inaccurate reprojection leads to ghosting or blurring. The blending weight must be tuned to balance noise reduction against motion artifacts [S1, unofficial].
- **Threshold Selection:** If `TRESHOLD` is too high, the reprojection position may be set too early (shallow in the cloud), leading to misalignment. If too low, it may be set too late or not at all if the ray exits the volume quickly [S1, unofficial].
- **Precision:** Floating-point precision in world-to-screen projection can cause jitter in the history buffer sampling if the camera moves significantly between frames [G].

## Operating characteristics
- **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of march steps. The reprojection step adds a constant overhead for matrix multiplication and buffer sampling.
- **Memory Complexity:** $O(W \times H)$ for the history buffer, where $W$ and $H$ are the screen dimensions.
- **Performance:** Allows for a significant reduction in per-frame sample counts compared to non-jittered methods, reducing computational cost while maintaining visual quality through temporal accumulation [S1, unofficial].
- **Strengths:** Effective at reducing banding artifacts from low sample counts; leverages temporal coherence to smooth noise.
- **Limitations:** Introduces temporal latency and potential ghosting artifacts, especially with fast camera motion or dynamic volumetric changes. Requires a history buffer and reprojection logic.
- **When to use:** When rendering static or slowly moving volumetric scenes (e.g., clouds) where temporal coherence is high and banding from low sample counts is a concern.
- **When not to use:** In scenes with rapid camera motion or highly dynamic volumetric changes where ghosting artifacts would be distracting.

## Background
This technique is described in the context of volumetric cloud rendering, specifically as an adaptive sampling technique to reduce the cost of high-quality cloud rendering by lowering sample counts and using temporal accumulation to mitigate noise [S1, unofficial]. It is attributed to Jack Tollenaar in an unofficial article on volumetric cloud rendering [S1, unofficial].

## Sources
[S1] Volumetric Cloud Rendering - Jack Tollenaar, https://www.jacktollenaar.top/articles/clouds.html, fetched 2026-10-29 (unofficial)
