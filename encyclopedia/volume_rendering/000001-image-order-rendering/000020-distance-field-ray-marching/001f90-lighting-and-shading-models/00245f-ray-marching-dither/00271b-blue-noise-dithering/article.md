# Blue Noise Dithering

Also called: Blue Noise Ray Offset, Temporal Blue Noise Dithering

## Overview
Blue Noise Dithering is a technique to reduce visible banding artifacts in volumetric ray marching by introducing a random offset to the ray step start position, derived from a blue noise texture, allowing for larger step sizes without quality loss [S1, S2, unofficial]. It sets itself apart from [Ray Marching Jittering](5399-ray-marching-jittering.md) by using a structured blue noise pattern rather than uniform or white noise, which minimizes clumping and produces a less perceptible error pattern, and from its parent [Ray Marching Dither](3269-ray-marching-dither.md) by specifically targeting the initialization offset of the ray march loop to enable significant performance gains through increased step sizes [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `uBlueNoise`: A 2D sampler texture containing a pre-generated blue noise pattern [S1, S2, unofficial].
    *   `frame`: The current frame index (integer) [S2, unofficial].
    *   `rayOrigin`: The starting position of the ray (vec3) [S1, S2, unofficial].
    *   `rayDir`: The direction of the ray (vec3) [S1, S2, unofficial].
    *   `MARCH_SIZE` / `STEP_SIZE`: The distance advanced per raymarching step [S1, S2, unofficial]. Note: [S1, unofficial] uses `MARCH_SIZE` while [S2, unofficial] uses `STEP_SIZE`; they represent the same parameter.
    *   `NUM_STEPS`: The maximum number of raymarching iterations [S1, S2, unofficial].
*   **Outputs:**
    *   The result of the raymarching loop, typically a color or density value [S1, S2, unofficial].

**Data Structures:**
*   **Blue Noise Texture:** A 2D texture (typically 1024x1024 or similar power-of-two resolution) where the red channel contains values in the range [0, 1) [S1, S2, unofficial].

**Parameters:**
*   **Typical Values:**
    *   `NUM_STEPS`: Reduced from 250 to 50 in optimized implementations [S2, unofficial].
    *   `STEP_SIZE`: Increased from 0.05 to 0.5 in optimized implementations [S2, unofficial].
    *   **Performance Impact:** This configuration reduces total iterations from 5000 to 100 while maintaining similar output quality [S2, unofficial].
    *   **Temporal Shift:** The frame-dependent shift uses `frame % 32` and divides by `sqrt(0.5)` [S1, S2, unofficial].

**Numbered Steps:**
1.  **Sample Blue Noise:** Sample the blue noise texture at normalized fragment coordinates. The coordinates are calculated as `gl_FragCoord.xy / 1024.0`. Extract the red channel value [S1, S2, unofficial].
2.  **Calculate Temporal Offset:** Add a frame-dependent shift to the sampled blue noise value to ensure temporal variation. The shift is calculated as `float(frame % 32) / sqrt(0.5)` [S1, S2, unofficial].
3.  **Compute Final Offset:** Apply the `fract` function to the sum of the blue noise value and the temporal shift to keep the result in the range [0, 1). This value serves as the dithering offset [S1, S2, unofficial].
4.  **Initialize Ray Distance:** Calculate the initial ray march distance `t` by multiplying the `STEP_SIZE` (or `MARCH_SIZE`) by the computed offset [S1, S2, unofficial].
5.  **Execute Ray Marching Loop:** Iterate `NUM_STEPS` times. In each iteration, calculate the sample position using `rayOrigin + rayDir * t`. Advance `t` by `STEP_SIZE` (or `MARCH_SIZE`) in subsequent steps [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blue_noise_dithering(rayOrigin, rayDir, uBlueNoise, frame, STEP_SIZE, NUM_STEPS):
    # Step 1: Sample blue noise texture
    # Coordinates normalized to [0, 1) assuming 1024x1024 texture
    uv = gl_FragCoord.xy / 1024.0
    blueNoise = texture2D(uBlueNoise, uv).r

    # Step 2 & 3: Calculate temporal offset and apply fract
    # Temporal shift ensures the pattern changes every frame
    temporalShift = float(frame % 32) / sqrt(0.5)
    offset = fract(blueNoise + temporalShift)

    # Step 4: Initialize ray distance
    t = STEP_SIZE * offset

    # Step 5: Execute ray marching loop
    for i in range(NUM_STEPS):
        # Calculate sample position
        samplePos = rayOrigin + rayDir * t
        
        # Perform shading/sampling logic here (e.g., density accumulation)
        # ...
        
        # Advance ray distance
        t += STEP_SIZE

    return result
```

**Edge Cases and Pitfalls:**
*   **Noticeable Pattern:** The dithering pattern can be noticeable in single frames if only spatial noise is used. This is mitigated by adding the temporal component based on the frame count [S1, S2, unofficial].
*   **Banding Artifacts:** Visible banding occurs if the step size is increased without dithering. The blue noise offset resolves this by distributing the sampling error [S2, unofficial].
*   **Precision:** The use of `fract` ensures the offset remains bounded, preventing the ray from starting outside the intended volume or causing negative distances if not handled correctly [G].

## Subtopics

### Spatial Blue Noise Dithering
Uses only the spatial blue noise texture without temporal variation. Suitable for static scenes or when frame coherence is not required. [Link](#)

### Temporal Blue Noise Dithering
Adds a frame-dependent shift to the blue noise value. Recommended for animated scenes to prevent static pattern visibility. [Link](#)

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of steps. The dithering calculation is $O(1)$ per pixel.
*   **Memory Complexity:** $O(1)$ for the dithering offset calculation, plus the memory cost of the blue noise texture (typically small, e.g., 1024x1024x4 bytes) [S1, S2, unofficial].
*   **Measured Performance:** In the cited implementation, reducing iterations from 5000 to 100 (by increasing step size from 0.05 to 0.5 and steps from 250 to 50) resulted in a significant performance improvement while maintaining similar output quality [S2, unofficial].
*   **Strengths:** Allows for larger step sizes, reducing the number of samples needed per ray. Reduces visible banding artifacts effectively.
*   **Limitations:** The dithering pattern may be visible in single frames without temporal variation. Requires a pre-generated blue noise texture [S1, S2, unofficial].
*   **When to Use:** When ray marching volumetric effects (clouds, fog, light shafts) and performance is a concern, and banding artifacts are visible with standard step sizes.
*   **When Not to Use:** When high-quality single-frame stills are required without temporal averaging, or when the blue noise texture is not available.

## Background
Blue Noise Dithering in the context of volumetric ray marching was popularized by Maxime Heckel in his blog posts on real-time volumetric rendering [S1, S2, unofficial]. The technique leverages the properties of blue noise, which has a power spectrum that is concentrated at higher frequencies, making it less visible to the human eye compared to white or pink noise [S1, unofficial]. This allows for the introduction of random offsets that disrupt regular sampling patterns without introducing noticeable noise artifacts.

## Sources
[S1] Real-time dreamy Cloudscapes with Volumetric Raymarching, https://blog.maximeheckel.com/posts/real-time-cloudscapes-with-volumetric-raymarching/, fetched 2026-10-28 (unofficial)
[S2] On Shaping Light: Real-Time Volumetric Lighting with Post-Processing ..., https://blog.maximeheckel.com/posts/shaping-light-volumetric-lighting-with-post-processing-and-raymarching/, fetched 2026-10-28 (unofficial)
