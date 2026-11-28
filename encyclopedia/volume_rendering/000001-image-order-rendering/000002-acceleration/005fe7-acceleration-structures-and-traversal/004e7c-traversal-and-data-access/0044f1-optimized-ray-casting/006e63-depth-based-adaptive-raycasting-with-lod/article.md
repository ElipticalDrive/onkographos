# Depth-based Adaptive Raycasting with LOD

Also called: EA-raycasting, EA-raycasting with LOD and Depth-based Step Size, Front-to-Back Volume Raycasting with Early Termination and LOD, RaycastResult raycast, simpleRaycastLodDepth, simpleRaycastLodDepthOcclusion

## Overview
Depth-based Adaptive Raycasting with LOD is a front-to-back volume rendering method that optimizes performance by varying the sampling step size based on the ray's depth from the camera, combined with early ray termination [S1, unofficial]. It distinguishes itself from fixed-step raycasting by reducing computational cost in distant regions where visual detail is less critical, while maintaining high resolution near the viewer [S1, unofficial]. It builds on [Optimized Ray Casting](11373-optimized-ray-casting.md) by introducing depth-aware LOD selection.

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `passUV`: Screen-space UV coordinates of the current fragment [S1, unofficial].
    *   `front_uvw_map`, `back_uvw_map`: Textures storing the start (`startUVW`) and end (`endUVW`) 3D coordinates of the ray in volume space [S1, unofficial].
    *   `volume_texture`: The 3D volume texture containing scalar values [S1, unofficial].
    *   `transferFunction`: A function mapping scalar values and step size to RGBA color [S1, unofficial].
    *   `stepSize`: Base sampling step size in world/view space [S1, unofficial].
    *   `uLodBegin`: Depth threshold where LOD degradation begins [S1, unofficial].
    *   `uLodRange`: Depth range over which LOD degrades from 0 to `uLodMaxLevel` [S1, unofficial].
    *   `uLodMaxLevel`: Maximum LOD level multiplier [S1, unofficial].
    *   `uScreenToTexture`: Matrix to convert screen coordinates to texture coordinates [S1, unofficial].
    *   `getViewCoord`: Function to linearize depth from view space [S1, unofficial].
    *   `uProjection`, `uViewToTexture`: Matrices for depth projection [S1, unofficial].
*   **Outputs:**
    *   `fragColor`: The final accumulated RGBA color [S1, unofficial].
    *   `fragFirstHit`: The projected depth of the first opaque sample [S1, unofficial].
    *   `gl_FragDepth`: The depth value for the fragment [S1, unofficial].

**Data Structures:**
*   `RaycastResult`: Struct containing:
    *   `color`: `vec4` (accumulated RGBA) [S1, unofficial].
    *   `firstHit`: `vec4` (xyz: position of first hit, a: depth of first hit) [S1, unofficial].
    *   `lastHit`: `vec4` (xyz: position of last hit, a: depth of last hit) [S1, unofficial].
*   `VolumeSample`: Struct containing:
    *   `value`: `float` (scalar value from texture) [S1, unofficial].
    *   `uvw`: `vec3` (sampling position) [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Initialization:**
    *   Retrieve `startUVW` and `endUVW` from `front_uvw_map` and `back_uvw_map` using `passUV` [S1, unofficial].
    *   **Edge Case (Invalid Pixel):** If `endUVW.a == 0.0` (ray does not intersect volume), discard the fragment [S1, unofficial].
    *   **Edge Case (Camera Inside Volume):** If `startUVW.a == 0.0`, clamp the start point to the near plane:
        ```python
        uvwStart = uScreenToTexture * vec4(passUV, 0.0, 1.0)
        uvwStart.a = 0.0
        startUVW = uvwStart
        ```
        [S1, unofficial]
    *   Linearize depths:
        ```python
        startDistance = abs(getViewCoord(vec3(passUV, startUVW.a)).z)
        endDistance = abs(getViewCoord(vec3(passUV, endUVW.a)).z)
        startDepth = startDistance
        endDepth = endDistance
        ```
        [S1, unofficial]
    *   Initialize `result`:
        ```python
        result.color = vec4(0.0)
        result.firstHit = vec4(startUVW, endDepth) # Placeholder
        result.lastHit = vec4(endUVW, endDepth)
        curColor = vec4(0.0)
        ```
        [S1, unofficial]
    *   Calculate parametric step size:
        ```python
        rayLength = length(endUVW - startUVW)
        parameterStepSize = stepSize / rayLength
        ```
        [S1, unofficial]
    *   Initialize ray parameter `t`:
        ```python
        t = 0.01
        if RANDOM_OFFSET is defined:
            t = t * 2.0 * rand(passUV)
        ```
        [S1, unofficial]

2.  **Ray Marching Loop:**
    *   Loop while `t < 1.0 + (0.5 * parameterStepSize)` [S1, unofficial]:
        1.  **Interpolate Position:**
            ```python
            curUVW = mix(startUVW, endUVW, t)
            curDepth = mix(startDepth, endDepth, t)
            ```
            [S1, unofficial]
        2.  **Calculate LOD Level:**
            ```python
            lodFactor = max(0.0, min(1.0, (curDepth - uLodBegin) / uLodRange))
            curLod = lodFactor * uLodMaxLevel
            ```
            [S1, unofficial]
        3.  **Calculate Adaptive Step Size:**
            ```python
            curStepSize = stepSize * pow(2.0, curLod)
            parameterStepSize = curStepSize / rayLength
            ```
            [S1, unofficial]
        4.  **Sample Volume:**
            ```python
            rawValue = textureLod(volume_texture, curUVW, curLod).r
            curSample.value = int(rawValue)
            curSample.uvw = curUVW
            ```
            [S1, unofficial]
        5.  **Transfer Function:**
            ```python
            sampleColor = transferFunction(curSample.value, curStepSize)
            ```
            [S1, unofficial]
        6.  **Record First Hit:**
            If `result.firstHit.a` is not set (or effectively 0) AND `sampleColor.a > 0.001`:
            ```python
            result.firstHit.rgb = curSample.uvw
            result.firstHit.a = curDepth
            ```
            [S1, unofficial]
        7.  **Front-to-Back Blending:**
            ```python
            curColor.rgb = (1.0 - curColor.a) * sampleColor.rgb + curColor.rgb
            curColor.a = (1.0 - curColor.a) * sampleColor.a + curColor.a
            ```
            [S1, unofficial]
        8.  **Early Ray Termination:**
            If `curColor.a > 0.99`, break loop [S1, unofficial].
        9.  **Increment Parameter:**
            ```python
            t += parameterStepSize
            ```
            [S1, unofficial]

3.  **Finalization:**
    *   Set `result.color = curColor` [S1, unofficial].
    *   Output `fragColor = result.color` [S1, unofficial].
    *   **Depth Output:**
        If `result.firstHit.a > 0.0`:
        ```python
        firstHitProjected = uProjection * inverse(uViewToTexture) * vec4(result.firstHit.xyz, 1.0)
        fragFirstHit.a = max((firstHitProjected.z / firstHitProjected.w) * 0.5 + 0.5, 0.0)
        gl_FragDepth = fragFirstHit.a
        ```
        Else:
        ```python
        fragFirstHit = endUVW
        gl_FragDepth = 1.0
        ```
        [S1, unofficial]

**Parameters:**
*   `stepSize`: Base sampling step size. Typical value: `1.0 / (2.0 * volume_size_x)` [S1, unofficial].
*   `uLodBegin`: Depth threshold for LOD start. Not given in the sources.
*   `uLodRange`: Depth range for LOD transition. Not given in the sources.
*   `uLodMaxLevel`: Maximum LOD multiplier. Not given in the sources.
*   `ALPHA_SCALE`: Multiplier for alpha in transfer function, default 20.0 [S1, unofficial].
*   `uWindowingRange`, `uWindowingMinVal`: Windowing parameters for transfer function [S1, unofficial].

**Pitfalls:**
*   **Camera Inside Volume:** Must clamp start to near plane [S1, unofficial].
*   **Invalid Rays:** Discard if `endUVW.a == 0.0` [S1, unofficial].
*   **Occlusion Culling:** If `uvwOcclusion.x` is between `startUVW.a` and `endUVW.a`, update `startUVW` to the occlusion point [S1, unofficial].
*   **Scene Clipping:** If `scene_depth < startUVW.a`, set `gl_FragDepth` to `startUVW.a` and return. If `scene_depth < endUVW.a`, update `endUVW` to `scene_depth` [S1, unofficial].
*   **Loop Bound:** The condition `t < 1.0 + (0.5 * parameterStepSize)` ensures the endpoint is covered despite variable steps [S1, unofficial].
*   **First Hit Logic:** `result.firstHit.a > curDepth` ensures the closest hit is recorded, assuming depth increases away from camera [S1, unofficial].

## Subtopics

### [Multi-layer Beer-Lambert Raycasting with Ambient Occlusion and Shadowing](21578-multi-layer-beer-lambert-raycasting-with-ambient-occlusion-a.md)
A GPU-based technique that splits rays into layers based on opacity thresholds, applying Beer-Lambert law and ambient occlusion. It builds on this method by using its LOD and variable step size to optimize sampling density within each layer.

## Operating characteristics
*   **Time Complexity:** Reduced compared to fixed-step raycasting due to larger step sizes at greater depths (higher LOD) [S1, unofficial].
*   **Memory Complexity:** Requires storage for `front_uvw_map` and `back_uvw_map` textures, plus the volume texture [S1, unofficial].
*   **Strengths:** Optimizes performance by adapting sampling density to depth; early termination further reduces cost for opaque regions [S1, unofficial].
*   **Limitations:** Visual quality may degrade in distant regions if LOD is too aggressive; requires pre-computed UVW maps [S1, unofficial].
*   **When to Use:** Real-time volume rendering where performance is critical and depth-based detail variation is acceptable [S1, unofficial].
*   **When Not to Use:** Applications requiring uniform high resolution throughout the volume or where depth-based LOD artifacts are unacceptable [S1, unofficial].

## Background
This method is implemented in the VRVRVive project by Secumfex [S1, unofficial]. It represents an evolution from simple fixed-step raycasting (`simpleRaycast`) by introducing depth-based LOD and step size adjustment [S1, unofficial].

## Sources
[S1] Secumfex/VRVRVive, https://github.com/Secumfex/VRVRVive, fetched 2026-10-28 (unofficial)
