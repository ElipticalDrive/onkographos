# Monte Carlo Path Tracing for Volume Rendering

Also called: MCSRender, Monte Carlo Scattering Rendering

## Overview
A volume rendering technique that uses Monte Carlo path tracing to simulate light scattering within participating media, calculating transmittance and accumulating results over multiple frames to reduce noise [S1, unofficial]. It distinguishes itself from deterministic ray marching by using stochastic distance sampling and exponential moving average accumulation, trading convergence speed for unbiased scattering behavior [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs**: Screen position (`vPosition`), MVP inverse matrix, volume transfer function (for alpha/color), environment map (for lighting), and uniform parameters (`uSigmaMax`, `uAlphaCorrection`, `uScatteringDirection`, `uOffset`, `uInvFrameNumber`).
*   **Outputs**: Accumulated color (`oColor`) in a framebuffer texture.

**Data Structures**
*   **Accumulator Texture**: Stores the running average of rendered frames.
*   **Frame Texture**: Stores the current frame's contribution before accumulation.
*   **Uniforms**:
    *   `uSigmaMax` (float): Maximum scattering coefficient for distance sampling.
    *   `uAlphaCorrection` (float): Correction factor for alpha values from the transfer function.
    *   `uScatteringDirection` (vec3): Direction for secondary ray scattering.
    *   `uOffset` (float): Offset for random seed generation.
    *   `uInvFrameNumber` (float): Inverse of the current frame number for accumulation.

**Algorithm Steps**

1.  **Ray Initialization**: Unproject the screen position to obtain the ray origin `vRayFrom` and ray end `vRayTo` using the inverse Model-View-Projection matrix.
2.  **Bounding Box Intersection**: Intersect the ray with the volume bounding cube to get `tbounds` (near and far t-values).
    *   If `tbounds.x >= tbounds.y`, the ray misses the volume. Sample the environment map and output as color.
3.  **Segment Calculation**: Calculate the ray segment `from` and `to` within the cube. Compute `maxDistance` as the length of this segment.
4.  **Seed Initialization**: Initialize a random seed: `seed = vPosition + rand(vec2(uOffset, uOffset))`.
5.  **Distance Sampling**: Call `sampleDistance(from, to, seed)` to find the first scattering distance `dist`.
    *   If `dist > maxDistance`, sample the environment map and output as color.
6.  **Update Position**: Update the ray origin to the scattering point: `from = mix(from, to, dist / maxDistance)`.
7.  **Secondary Ray Intersection**: Intersect a new ray starting at `from` with direction `uScatteringDirection` to get new `tbounds` and `to`.
8.  **Color Sampling**:
    *   Sample diffuse color at `from`: `diffuseColor = sampleVolumeColor(from)`.
    *   Sample light color from environment: `lightColor = sampleEnvironmentMap(uScatteringDirection)`.
9.  **Transmittance Calculation**: Calculate transmittance from `from` to `to` using `sampleTransmittance(from, to, seed)`.
10. **Final Color**: Compute the pixel color: $oColor = diffuseColor \times lightColor \times transmittance$.
11. **Accumulation**: Update the accumulator using an exponential moving average:
    $$acc = texture(uAccumulator, vPosition)$$
    $$frame = texture(uFrame, vPosition)$$
    $$oColor = acc + (frame - acc) \times uInvFrameNumber$$
12. **Reset**: To reset the accumulation, set `oColor = vec4(0.0, 0.0, 0.0, 1.0)`.

**Subroutines**

*   `sampleDistance(from, to, seed)`:
    1.  Initialize `dist = 0.0`, `invSigmaMax = 1.0 / uSigmaMax`, `invMaxDistance = 1.0 / maxDistance`.
    2.  Loop:
        a. Generate random `seed`.
        b. Update distance: `dist -= log(1.0 - seed.x) * invSigmaMax`.
        c. If `dist > maxDistance`, break and return `dist`.
        d. Calculate sampling position: `samplingPosition = mix(from, to, dist * invMaxDistance)`.
        e. Sample volume alpha: `alphaSample = transferSample.a * uAlphaCorrection`.
        f. If `seed.y < alphaSample * invSigmaMax`, break and return `dist`.

*   `sampleTransmittance(from, to, seed)`:
    1.  Initialize `transmittance = 1.0`.
    2.  Loop similarly to `sampleDistance`:
        a. Generate random `seed`.
        b. Update distance: `dist -= log(1.0 - seed.x) * invSigmaMax`.
        c. If `dist > maxDistance`, break.
        d. Calculate sampling position: `samplingPosition = mix(from, to, dist * invMaxDistance)`.
        e. Sample volume alpha: `alphaSample = transferSample.a * uAlphaCorrection`.
        f. Update transmittance: `transmittance *= 1.0 - alphaSample * invSigmaMax`.
    3.  Return `transmittance`.

**Parameters**
*   `uSigmaMax`: Typical value depends on the medium's density; acts as the upper bound for the exponential distribution of scattering distances.
*   `uAlphaCorrection`: Scales the alpha value from the transfer function to match the physical scattering properties.
*   `uInvFrameNumber`: Decreases as frames accumulate, stabilizing the image.

**Edge Cases and Pitfalls**
*   **Precision**: Logarithms of values close to 1.0 can suffer from precision loss; ensure `1.0 - seed.x` is clamped to a minimum epsilon.
*   **Boundaries**: If `tbounds.x >= tbounds.y`, the ray does not intersect the volume, and environment sampling must be handled correctly.
*   **Degenerate Input**: If `uSigmaMax` is zero, division by zero occurs in `invSigmaMax`.

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of scattering events per ray, which is stochastic.
*   **Memory Complexity**: Requires storage for the accumulator and frame textures, proportional to screen resolution.
*   **Performance**: Converges slowly due to Monte Carlo noise, requiring multiple frames. Hardware performance depends on GPU fragment shader throughput and texture sampling efficiency.
*   **Strengths**: Unbiased scattering, handles complex lighting and shadows within volumes naturally.
*   **Limitations**: High noise levels in early frames, slow convergence, sensitive to `uSigmaMax` selection.
*   **When to use**: When unbiased scattering is required and temporal accumulation is acceptable.
*   **When not to use**: When real-time performance without accumulation is needed, or for simple homogeneous volumes where analytical solutions exist.

## Background
Originated in the context of GPU-based volume rendering, specifically implemented in the `vpt-conductor` project by UL-FRI-LGM [S1, unofficial]. It represents an approach to scattering rendering that leverages Monte Carlo methods for distance sampling and transmittance calculation, accumulating results over time to achieve a noise-reduced image.

## Sources
[S1] UL-FRI-LGM/vpt-conductor, https://github.com/UL-FRI-LGM/vpt-conductor, fetched 2026-10-26 (unofficial)
