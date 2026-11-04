# Volumetric Ray Marching with Noise

Also called: Noise Volume Rendering

## Overview
A real-time volumetric rendering technique that samples a 3D noise texture along a ray passing through a spherical cloud volume to compute opacity and color [S1, unofficial]. It differs from its parent [Fixed-Step Raymarching with Beer's Law](2452-fixed-step-raymarching-with-beer-s-law.md) by using a normalized noise vector to modulate density and light accumulation, rather than relying solely on scalar density and Beer's Law attenuation [S1, unofficial].

## How it works
**Inputs:**
- `fragPos`: Fragment position in world space.
- `viewRay`: Normalized view direction vector.
- `center`: Center of the spherical cloud volume.
- `radius`: Radius of the spherical cloud volume.
- `fNoiseSizeAdjust`: Scaling factor to convert world space to local texture space.
- `numOctaves`: Number of octaves for the noise function.
- `noiseOpacity`: Scaling factor for accumulated opacity.
- `minNoiseSteps`, `maxNoiseSteps`: Clamping bounds for step count.
- `minNoiseColor`: Base color for the noise volume.
- `noiseColorScale`: Scaling factor for the noise color contribution.

**Outputs:**
- `color`: Final RGBA color of the fragment.

**Data Structures:**
- `noise3D(pos, octaves)`: A 3D noise function returning a `vec4` (or `vec3` with alpha) where `.xyz` represents the noise vector and `.a` represents the noise magnitude/alpha.
- `saturate(x)`: Clamps `x` to `[0, 1]`.
- `normalize(v)`: Normalizes vector `v`.
- `dot(a, b)`: Dot product of vectors `a` and `b`.
- `length(v)`: Euclidean length of vector `v`.
- `vec2(0.5)`: Constant vector for alpha calculation.

**Steps:**

1.  **Ray-Sphere Intersection:**
    Compute the intersection of the ray `fragPos + t * viewRay` with the sphere defined by `center` and `radius`. Obtain `tnear` and `tfar` (the smallest and largest positive `t` values). If no intersection exists, discard the fragment.

2.  **World Space Bounds:**
    Calculate the world-space start and end points of the ray segment inside the sphere:
    $$ \text{worldNear} = \text{fragPos} + \text{viewRay} \times \text{tnear} $$
    $$ \text{worldFar} = \text{fragPos} + \text{viewRay} \times \text{tfar} $$

3.  **Local Texture Coordinates:**
    Convert world-space points to local texture coordinates:
    $$ \text{localTexNear} = \text{worldNear} \times \text{fNoiseSizeAdjust} $$
    $$ \text{localTexFar} = \text{worldFar} \times \text{fNoiseSizeAdjust} $$

4.  **Step Count Calculation:**
    Calculate the number of steps based on the distance in texture space:
    $$ \text{iSteps} = \frac{\|\text{localTexFar} - \text{localTexNear}\|}{\text{stepSize}} $$
    Clamp `iSteps` to the range `[minNoiseSteps, maxNoiseSteps]`. Note: `stepSize` is an implicit parameter derived from the desired resolution or fixed value in the implementation context [G].

5.  **Initialization:**
    Initialize the marching variables:
    $$ \text{currentTex} = \text{localTexNear} $$
    $$ \text{localTexDelta} = \frac{\text{localTexFar} - \text{localTexNear}}{\text{iSteps} - 1} $$
    $$ \text{opacityAdjust} = \frac{\text{noiseOpacity}}{\text{iSteps} - 1} $$
    $$ \text{lightAdjust} = \frac{1.0}{\text{iSteps} - 1} $$
    $$ \text{runningOpacity} = 0 $$
    $$ \text{runningLight} = 0 $$
    $$ \text{unitTex} = \text{localTexNear} $$

6.  **Marching Loop:**
    Iterate `i` from `0` to `iSteps - 1`:
    a. Sample the noise:
       $$ \text{noiseCell} = \text{noise3D}(\text{currentTex}, \text{numOctaves}) $$
    b. Adjust noise by normal:
       $$ \text{noiseCell.xyz} += \text{normalize}(\text{unitTex}) $$
    c. Accumulate opacity:
       $$ \text{runningOpacity} += \text{noiseCell.a} \times (1.0 - \text{dot}(\text{unitTex}, \text{unitTex})) $$
    d. Accumulate light:
       $$ \text{runningLight} += \text{saturate}(\text{dot}(\text{noiseCell.xyz}, \text{vec3}(0, 1, 0)) \times 0.5 + 0.5) $$
    e. Advance texture coordinates:
       $$ \text{currentTex} += \text{localTexDelta} $$
       $$ \text{unitTex} += \text{localTexDelta} $$

7.  **Final Color and Alpha Computation:**
    $$ \text{col} = \text{minNoiseColor} + (\text{noiseColorScale} \times \text{runningLight} \times \text{lightAdjust}) $$
    $$ \text{alpha} = 1 - \|\text{fragTex} - \text{vec2}(0.5)\| \times 2 $$
    $$ \text{runningOpacity} = \text{saturate}(\text{runningOpacity} \times \text{opacityAdjust}) $$
    $$ \text{color} = \text{vec4}(\text{col}, \text{runningOpacity} \times \text{alpha}) $$

**Parameters:**
- `noiseOpacity`: Scaling factor for accumulated opacity.
- `minNoiseSteps`: Minimum number of steps.
- `maxNoiseSteps`: Maximum number of steps.
- `minNoiseColor`: Base color for the noise volume.
- `noiseColorScale`: Scaling factor for the noise color contribution.

**Edge Cases and Pitfalls:**
- Division by zero in `localTexDelta` and adjustments if `iSteps` is 1. The clamping of `minNoiseSteps` should ensure `iSteps >= 2` to avoid this, or a guard is required [G].
- `dot(unitTex, unitTex)` in opacity accumulation can exceed 1.0 if `unitTex` is not normalized, leading to negative opacity contributions. The source uses `unitTex` as a position vector, not a normalized direction, so this term acts as a radial falloff [S1, unofficial].
- `fragTex` in the alpha calculation is not defined in the source steps. It is assumed to be the texture coordinate of the fragment in screen or local space, or potentially `localTexNear` or `currentTex` at the end of the march. Given the context of `vec2(0.5)`, it likely refers to a 2D projection or screen-space coordinate [G].

## Operating characteristics
Not given in the sources.

## Background
This method is part of a procedural cloud rendering pipeline, specifically utilizing noise-based density modulation. It is implemented in the `Cloud-Renderer` project [S1, unofficial].

## Sources
[S1] jaafersheriff/Cloud-Renderer, https://github.com/jaafersheriff/Cloud-Renderer, fetched 2026-10-27 (unofficial)
