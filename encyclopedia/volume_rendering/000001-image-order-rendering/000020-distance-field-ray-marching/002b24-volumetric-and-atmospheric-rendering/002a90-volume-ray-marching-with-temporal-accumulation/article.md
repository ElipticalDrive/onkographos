# Volume Ray Marching with Temporal Accumulation

Also called: HZL Fluid Effects Volume Rendering

## Overview
Volume Ray Marching with Temporal Accumulation is a real-time volume rendering technique that marches rays through a 3D density field to accumulate color and opacity, enhanced by blending the current frame's result with a history texture [S1, unofficial]. This method solves the problem of noise and flickering in sparse or low-resolution volumetric data by leveraging temporal coherence, stabilizing the image over frames [S1, unofficial]. It is distinguished from standard ray marching by its explicit use of a history buffer and temporal blending weight, and from static SDF rendering by its reliance on sampled density textures rather than implicit distance functions [S1, unofficial].

## How it works
The method operates in a fragment shader, processing each pixel to compute a final color by marching through a 3D volume and accumulating contributions, then blending this result with the previous frame's output.

### Inputs and Outputs
**Inputs:**
*   `cameraPosition`: Camera origin in world space.
*   `inverseViewProjection`: Inverse of the view-projection matrix.
*   `textureCoordinate`: Fragment texture coordinate in $[0, 1]$.
*   `densityTexture`: 3D texture storing density values (sampled at `position * 0.5 + 0.5`).
*   `sceneDepthTexture`: Depth buffer of scene geometry (GL_DEPTH_COMPONENT24).
*   `historyTexture`: Previous frame's accumulated color (GL_RGBA8).
*   Uniforms: `stepSize`, `maximumSteps`, `frameIndex`, `jitterEnabled`, `temporalEnabled`, `historyWeight`, `fieldMode`, `lightDirection`, `lightIntensity`, `absorption`, `emissionStrength`, `selfShadowingEnabled`, `densityScale`.

**Outputs:**
*   `m_colorTexture`: Final rendered color (GL_RGBA8).

### Data Structures
*   **Textures**:
    *   `m_colorTexture`: GL_RGBA8, GL_LINEAR filtering, GL_CLAMP_TO_EDGE wrap.
    *   `m_depthTexture`: GL_DEPTH_COMPONENT24, GL_NEAREST filtering, GL_CLAMP_TO_EDGE wrap.
    *   `m_sceneDepthTexture`: GL_DEPTH_COMPONENT24, GL_NEAREST filtering, GL_CLAMP_TO_EDGE wrap.
    *   `m_historyTexture`: GL_RGBA8, GL_LINEAR filtering, GL_CLAMP_TO_EDGE wrap. Cleared to zero at initialization.
*   **Ray Intersection**: A `vec2` storing `[t_min, t_max]` for the ray-box intersection.

### Numbered Steps

1.  **Ray Initialization**:
    *   Convert `textureCoordinate` to clip space: `clipPosition = textureCoordinate * 2.0 - 1.0`.
    *   Unproject the far plane point: `farWorld = (inverseViewProjection * vec4(clipPosition.xy, 1.0, 1.0)).xyz / (inverseViewProjection * vec4(clipPosition.xy, 1.0, 1.0)).w`.
    *   Calculate ray direction: `rayDirection = normalize(farWorld - cameraPosition)`.
    *   Calculate inverse direction: `inverseDirection = 1.0 / rayDirection`. [G: Guard against division by zero if `rayDirection` is zero, though sources assume normalized valid directions].

2.  **Box Intersection**:
    *   Compute intersection with the unit cube $[-1, 1]^3$:
        *   `first = (-1.0 - cameraPosition) * inverseDirection`
        *   `second = (1.0 - cameraPosition) * inverseDirection`
        *   `minimum = min(first, second)`
        *   `maximum = max(first, second)`
        *   `intersection = vec2(max(minimum.x, minimum.y, minimum.z), min(maximum.x, maximum.y, maximum.z))`
    *   Set `distanceAlongRay = max(intersection.x, 0.0)`.

3.  **Scene Geometry Clipping**:
    *   Sample `sceneDepthTexture` at `textureCoordinate` to get `sceneDepth`.
    *   If `sceneDepth < 1.0`:
        *   Unproject scene point: `scenePoint = unproject(sceneDepth, textureCoordinate)`.
        *   Calculate `sceneDistance = dot(scenePoint - cameraPosition, rayDirection)`.
        *   Update `intersection.y = min(intersection.y, sceneDistance)`.
    *   If `intersection.y <= distanceAlongRay`, discard the fragment (ray does not intersect volume).

4.  **Jitter Application**:
    *   If `jitterEnabled`:
        *   Compute `jitter = random(gl_FragCoord.xy, frameIndex) * stepSize`.
        *   `distanceAlongRay += jitter`.
    *   The random function is defined as:
        ```python
        def random(coordinate, frame_index):
            value = fract(vec3(coordinate.xyx) * 0.1031)
            value += dot(value, value.yzx + 33.33 + float(frame_index) * 0.01)
            return fract((value.x + value.y) * value.z)
        ```

5.  **Accumulation Initialization**:
    *   `accumulated = vec4(0.0)`.
    *   `foundVolume = false`.

6.  **Ray Marching Loop**:
    *   Loop `index` from 0 to `maximumSteps - 1` (max 767 in source).
    *   Break if `index >= maximumSteps` or `distanceAlongRay >= intersection.y`.
    *   Calculate position: `position = cameraPosition + rayDirection * distanceAlongRay`.
    *   Sample density: `fieldValue = texture(densityTexture, position * 0.5 + 0.5).r`.
    *   Set `density = fieldValue`.

7.  **Volume Entry Detection**:
    *   If `!foundVolume` and `density > 0.01`:
        *   `representativePosition = position`.
        *   `foundVolume = true`.

8.  **Color and Lighting Computation**:
    *   Compute `lightTransmittance` if `selfShadowingEnabled`:
        *   March 20 steps in direction `-normalize(lightDirection + vec3(0.0001))` with step size 0.08.
        *   Accumulate `opticalDepth += texture(densityTexture, shadowPos * 0.5 + 0.5).r * shadowStep`.
        *   Break if `abs(shadowPos) > 1.0`.
        *   `lightTransmittance = exp(-opticalDepth * absorption * densityScale * 3.0)`.
    *   Compute `smokeColor` based on `fieldMode`:
        *   **Mode 0 (Smoke/Fire)**:
            *   Sample `temperature` and `fire` textures.
            *   `ambient = vec3(0.12, 0.15, 0.20)`.
            *   `direct = vec3(1.0, 0.88, 0.72) * lightIntensity * lightTransmittance`.
            *   Mix smoke color based on `position.y`.
            *   `flameColor` from temperature.
            *   `emittedLight = flameColor * fire * emissionStrength`.
        *   **Mode 1 (Simple Smoke)**:
            *   `smokeColor = mix(vec3(0.3, 0.04, 0.01), vec3(1.0, 0.65, 0.08), fieldValue)`.
        *   **Mode 2/3 (Abstract)**:
            *   `density = abs(fieldValue - 0.5) * 2.0`.
            *   `smokeColor = vec3(1.0, 0.0, 0.0)` if `fieldValue >= 0.5` else `vec3(0.0, 0.0, 1.0)`.

9.  **Accumulation (Implied)**:
    *   Although the source code snippet ends before the blending loop, the presence of `accumulated` and `temporalEnabled` implies standard volume rendering accumulation:
        ```python
        alpha = 1.0 - exp(-density * stepSize)
        color = smokeColor * alpha
        accumulated.rgb += color * (1.0 - accumulated.a)
        accumulated.a += alpha * (1.0 - accumulated.a)
        ```
    *   [G: This step is necessary to correctly use the `accumulated` variable and is standard for volume rendering. The source notes explicitly state this loop is expected.]

10. **Temporal Accumulation**:
    *   If `temporalEnabled`:
        *   `currentFrameColor = accumulated`.
        *   `historyColor = texture(historyTexture, textureCoordinate).rgb`.
        *   `finalColor = mix(currentFrameColor, historyColor, historyWeight)`.
    *   Else:
        *   `finalColor = accumulated.rgb`.

11. **Output**:
    *   Write `finalColor` to `m_colorTexture`.

### Parameters
*   `gridSpacing`: float, default 0.5. Debug visualization spacing.
*   `densityScale`: float. Multiplier for density in light transmittance.
*   `stepSize`: float. Distance advanced per step.
*   `maximumSteps`: int. Max steps per ray (up to 767).
*   `frameIndex`: int. Current frame number for jitter.
*   `jitterEnabled`: bool. Enable random offset.
*   `adaptiveSteppingEnabled`: bool. Uniform present, usage not shown in main loop.
*   `fieldMode`: int. 0: Smoke/Fire, 1: Simple Smoke, 2/3: Abstract.
*   `lightDirection`: vec3. Light source direction.
*   `lightIntensity`: float. Light intensity multiplier.
*   `absorption`: float. Absorption coefficient.
*   `emissionStrength`: float. Emission multiplier.
*   `selfShadowingEnabled`: bool. Enable shadow ray marching.
*   `temporalEnabled`: bool. Enable temporal accumulation.
*   `historyWeight`: float. Weight for previous frame blending.

### Edge Cases and Pitfalls
*   **Division by Zero**: `intersectBox` uses `1.0 / rayDirection`. If `rayDirection` is zero, this results in infinity/NaN. The code relies on the caller to provide valid normalized directions [S1, unofficial].
*   **Light Direction Normalization**: A small epsilon (0.0001) is added to `lightDirection` before normalization to prevent issues if the light direction is exactly zero [S1, unofficial].
*   **Density Texture Coordinate Mapping**: The density texture is sampled at `position * 0.5 + 0.5`, mapping the world space $[-1, 1]$ box to texture space $[0, 1]$ [S1, unofficial].
*   **No Volume Intersection**: If `intersection.y <= distanceAlongRay`, the fragment is discarded [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot M)$, where $N$ is the number of pixels and $M$ is the maximum number of steps per ray (`maximumSteps`). Self-shadowing adds a constant factor (20 steps) per hit.
*   **Memory Complexity**: $O(W \cdot H)$ for the color, depth, and history textures, where $W$ and $H$ are the viewport dimensions. The density texture is $O(V^3)$ for volume resolution $V$.
*   **Measured Performance**: The source provides performance metrics in FPS and milliseconds but does not specify the hardware, resolution, or scene details used for measurement [S1, unofficial].
*   **Strengths**: Reduces noise and stabilizes the image over time using temporal accumulation. Allows for lower sampling rates (fewer steps) by leveraging history.
*   **Limitations**: Requires a history texture and temporal blending, which can introduce ghosting if the camera or volume moves quickly. Performance is limited by the number of steps and texture sampling.
*   **When to Use**: Real-time volumetric rendering where noise reduction is critical and motion is relatively smooth.
*   **When Not to Use**: Scenes with rapid camera movement or high-frequency volume changes, where ghosting artifacts would be distracting.

## Background
This method is implemented in the HZL Fluid Effects Studio, an open-source fluid simulation and visualization tool [S1, unofficial]. It is part of the HZL (Hazel) graphics library ecosystem. The technique combines standard volume ray marching with temporal accumulation, a common approach in real-time graphics to improve image quality without increasing per-frame computational cost.

## Sources
[S1] pvasbob/HZL-fluidEffectsStudio, https://github.com/pvasbob/HZL-fluidEffectsStudio, fetched 2026-10-27 (unofficial)
