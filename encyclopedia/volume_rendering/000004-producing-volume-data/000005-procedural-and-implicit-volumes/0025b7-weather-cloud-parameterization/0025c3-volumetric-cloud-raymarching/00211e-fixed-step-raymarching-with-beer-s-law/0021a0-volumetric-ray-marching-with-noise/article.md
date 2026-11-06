# Volumetric Ray Marching with Noise

Also called: Noise Volume Rendering

## Overview
A real-time volumetric rendering technique that samples a 3D noise texture along a ray passing through a spherical cloud volume to compute opacity and color [S1, unofficial].

## How it works
**Inputs:**
- `fragPos`: Fragment position in world space [S1, unofficial].
- `viewRay`: Normalized view direction vector [S1, unofficial].
- `center`: Center of the spherical cloud volume [S1, unofficial].
- `radius`: Radius of the spherical cloud volume [S1, unofficial].
- `fNoiseSizeAdjust`: Scaling factor to convert world space to local texture space [S1, unofficial].
- `numOctaves`: Number of octaves for the noise function [S1, unofficial].
- `noiseOpacity`: Scaling factor for accumulated opacity [S1, unofficial].
- `minNoiseSteps`: Minimum number of steps [S1, unofficial].
- `maxNoiseSteps`: Maximum number of steps [S1, unofficial].
- `minNoiseColor`: Base color for the noise volume [S1, unofficial].
- `noiseColorScale`: Scaling factor for the noise color contribution [S1, unofficial].
- `stepSize`: Step size in texture space [G].

**Outputs:**
- `color`: Final RGBA color of the fragment [S1, unofficial].

**Data Structures:**
- `noise3D(pos, octaves)`: A 3D noise function returning a `vec4` where `.xyz` represents the noise vector and `.a` represents the noise magnitude/alpha [S1, unofficial].
- `saturate(x)`: Clamps `x` to `[0, 1]` [G].
- `normalize(v)`: Normalizes vector `v` [G].
- `dot(a, b)`: Dot product of vectors `a` and `b` [G].
- `length(v)`: Euclidean length of vector `v` [G].

**Steps:**
1. **Ray-Sphere Intersection:** Compute the intersection of the ray `fragPos + t * viewRay` with the sphere defined by `center` and `radius`. Obtain `tnear` and `tfar` (the smallest and largest positive `t` values). If no intersection exists, discard the fragment [S1, unofficial].
2. **World Space Bounds:** Calculate the world-space start and end points: `worldNear = fragPos + viewRay * tnear`, `worldFar = fragPos + viewRay * tfar` [S1, unofficial].
3. **Local Texture Coordinates:** Convert to local texture space: `localTexNear = worldNear * fNoiseSizeAdjust`, `localTexFar = worldFar * fNoiseSizeAdjust` [S1, unofficial].
4. **Step Count:** Calculate steps: `iSteps = length(localTexFar - localTexNear) / stepSize`. Clamp `iSteps` to `[minNoiseSteps, maxNoiseSteps]` [S1, unofficial].
5. **Initialization:** Set `currentTex = localTexNear`, `localTexDelta = (localTexFar - localTexNear) / (iSteps - 1)`, `opacityAdjust = noiseOpacity / (iSteps - 1)`, `lightAdjust = 1.0 / (iSteps - 1)`, `runningOpacity = 0`, `runningLight = 0`, `unitTex = localTexNear` [S1, unofficial].
6. **Marching Loop:** Iterate `iSteps` times: sample `noiseCell = noise3D(currentTex, numOctaves)`, adjust `noiseCell.xyz += normalize(unitTex)`, accumulate `runningOpacity += noiseCell.a * (1.0 - dot(unitTex, unitTex))`, accumulate `runningLight += saturate(dot(noiseCell.xyz, vec3(0, 1, 0)) * 0.5 + 0.5)`, advance `currentTex += localTexDelta`, advance `unitTex += localTexDelta` [S1, unofficial].
7. **Final Computation:** Compute `col = minNoiseColor + (noiseColorScale * runningLight * lightAdjust)`, `alpha = 1 - length(fragTex - vec2(0.5)) * 2`, `runningOpacity = saturate(runningOpacity * opacityAdjust)`, `color = vec4(vec3(col), runningOpacity * alpha)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_march_noise(fragPos, viewRay, center, radius, fNoiseSizeAdjust, numOctaves,
                    noiseOpacity, minNoiseSteps, maxNoiseSteps, minNoiseColor,
                    noiseColorScale, stepSize):
    tnear, tfar = ray_sphere_intersect(fragPos, viewRay, center, radius)
    if tnear is None: return vec4(0, 0, 0, 0) # Discard if no intersection [S1, unofficial]

    worldNear = fragPos + viewRay * tnear
    worldFar = fragPos + viewRay * tfar

    localTexNear = worldNear * fNoiseSizeAdjust
    localTexFar = worldFar * fNoiseSizeAdjust

    iSteps = length(localTexFar - localTexNear) / stepSize
    iSteps = clamp(iSteps, minNoiseSteps, maxNoiseSteps)
    if iSteps < 2: iSteps = 2 # Guard against division by zero [G]

    currentTex = localTexNear
    localTexDelta = (localTexFar - localTexNear) / (iSteps - 1)
    opacityAdjust = noiseOpacity / (iSteps - 1)
    lightAdjust = 1.0 / (iSteps - 1)
    runningOpacity = 0.0
    runningLight = 0.0
    unitTex = localTexNear # Initialized from source context [G]

    for _ in range(iSteps):
        noiseCell = noise3D(currentTex, numOctaves)
        noiseCell.xyz += normalize(unitTex)
        runningOpacity += noiseCell.a * (1.0 - dot(unitTex, unitTex))
        runningLight += saturate(dot(noiseCell.xyz, vec3(0, 1, 0)) * 0.5 + 0.5)
        currentTex += localTexDelta
        unitTex += localTexDelta

    col = minNoiseColor + (noiseColorScale * runningLight * lightAdjust)
    alpha = 1.0 - length(fragTex - vec2(0.5)) * 2.0 # fragTex undefined in source [G]
    runningOpacity = saturate(runningOpacity * opacityAdjust)
    return vec4(vec3(col), runningOpacity * alpha)
```

**Parameters:**
- `noiseOpacity`: Scaling factor for accumulated opacity [S1, unofficial].
- `minNoiseSteps`: Minimum number of steps [S1, unofficial].
- `maxNoiseSteps`: Maximum number of steps [S1, unofficial].
- `minNoiseColor`: Base color for the noise volume [S1, unofficial].
- `noiseColorScale`: Scaling factor for the noise color contribution [S1, unofficial].

**Edge Cases and Pitfalls:**
- Division by zero in `localTexDelta` and adjustments if `iSteps` is 1. The clamping of `minNoiseSteps` should ensure `iSteps >= 2`, or a guard is required [G].
- `dot(unitTex, unitTex)` in opacity accumulation grows quadratically as `unitTex` marches, potentially making the term `(1.0 - dot(unitTex, unitTex))` negative and reducing opacity contributions [S1, unofficial].
- `fragTex` in the alpha calculation is not defined in the source steps. It is assumed to be a 2D texture coordinate or screen-space projection [G].

## Operating characteristics
Not given in the sources.

## Background
This method is part of a procedural cloud rendering pipeline, specifically utilizing noise-based density modulation [S1, unofficial].

## Sources
[S1] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
