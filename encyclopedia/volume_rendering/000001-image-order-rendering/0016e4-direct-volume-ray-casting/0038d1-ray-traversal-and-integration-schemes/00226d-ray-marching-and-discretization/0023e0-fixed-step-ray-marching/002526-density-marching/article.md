# Density Marching

Also called: Simple Ray Marching

## Overview
Density Marching is a direct volume ray casting technique that samples a 3D texture along a ray cast from the camera through a volume bounding box, accumulating color and opacity using front-to-back blending [S1, unofficial]. It is distinguished from its siblings by its reliance on a fixed number of steps (`STEPS`) rather than adaptive or distance-based stepping, and its specific implementation of procedural noise (Worley and Perlin) combined with boundary fading for cloud-like volumes [S1, S2, unofficial]. Unlike [Ray Evaluation Stepping DVR](3461-ray-evaluation-stepping-dvr.md) which may use variable step sizes, Density Marching uses a uniform discretization of the ray path [S1, unofficial].

## How it works
**Inputs and Outputs:**
The method operates in a fragment shader. Inputs include a vertex attribute `vUVW` (vec3), uniform data containing `viewProj` (mat4) and `globalCameraPos` (vec4), a 3D texture sampler `volumeTex` (binding 1), and push constants containing `model` (mat4) and `localCameraPos` (vec4) [S1, unofficial]. The output is `outColor` (vec4) [S1, unofficial].

**Data Structures:**
*   `rayOrigin`: vec3, camera position.
*   `rayDirection`: vec3, normalized direction vector.
*   `accum`: vec4, accumulates RGB color and Alpha opacity.
*   `currentPos`: vec3, current position along the ray.
*   `deltaStep`: vec3, step vector.
*   `densityTotal`: float, integrated density for transmittance.
*   `brightness`: float, accumulated scattered light.

**Numbered Steps:**

1.  **Ray Direction Calculation:** Compute the ray direction by normalizing the difference between the interpolated vertex attribute and the local camera position:
    $$ \vec{rayDir} = \text{normalize}(\vec{vUVW} - \vec{localCameraPos}) $$
    [S1, unofficial]

2.  **Ray-Box Intersection:** Intersect the ray with the volume bounding box (defined as `boxMin = vec3(0.0)` and `boxMax = vec3(1.0)` in the source, though other sources use explicit min/max bounds [S2, unofficial]) using the slab method to obtain entry and exit distances `tNear` and `tFar` [S1, unofficial].
    [S1, unofficial]

3.  **Ray Segment Validation:** Calculate `tStart = max(tNear, 0.0)` and `tEnd = max(tFar, 0.0)`. Compute `totalDist = tEnd - tStart`. If `totalDist <= 0.0`, discard the fragment [S1, unofficial].

4.  **Ray Initialization:** Calculate the starting ray position at the box entry point:
    $$ \vec{rayPosition} = \vec{rayOrigin} + \vec{rayDir} \times tStart $$
    [S1, unofficial]
    Initialize `densityTotal = 0.0` and `brightness = 0.0` [S2, unofficial].

5.  **Wind Scrolling (Optional):** Apply wind scrolling to the ray position if time-based animation is enabled:
    $$ \vec{scrollPosition} = \vec{rayPosition} + (\vec{windTime}, 0, 0) $$
    [S2, unofficial]

6.  **Boundary Fading (Optional):** Calculate fade factors for the top and bottom boundaries to prevent abrupt cutoffs [S2, unofficial].
    $$ \text{distToBot} = \frac{\vec{rayPosition}.y - \text{boxMin}.y}{\text{boxMax}.y - \text{boxMin}.y} $$
    $$ \text{botFizzle} = \text{clamp}(\text{distToBot} \times 10.0, 0, 1) $$
    $$ \text{topFizzle} = \text{clamp}((1.0 - \text{distToBot}) \times 10.0, 0, 1) $$
    $$ \text{totalFizzle} = \text{botFizzle} \times \text{topFizzle} $$
    [S2, unofficial]

7.  **Step Calculation:** Compute the step size:
    $$ \text{stepSize} = \frac{\text{totalDist}}{\text{STEPS}} $$
    Initialize `currentPos = \vec{localCameraPos} + \vec{rayDir} \times tStart` and `deltaStep = \vec{rayDir} \times \text{stepSize}` [S1, unofficial].
    [S1, unofficial]

8.  **Procedural Density Generation (Optional):** If using procedural noise instead of a 3D texture, compute Worley density using Fractal Brownian Motion (FBM) [S2, unofficial]:
    *   Initialize `amplitude = 1.0`, `frequency = 1.0`, `amplitudeTotal = 0.0`, `worleyDensity = 0.0`.
    *   Loop `_OctaveCount` times:
        *   `worleyDensity += ProceduralWorley(\vec{scrollPosition} \times \text{frequency}, 1.0 / \text{CloudScale}) \times \text{amplitude}`
        *   `amplitudeTotal += amplitude`
        *   `amplitude /= 2.0`
        *   `frequency *= 2.0`
    *   `worleyDensity /= amplitudeTotal`
    [S2, unofficial]
    Additionally, compute Perlin density:
    $$ \text{perlinDensity} = \text{ProceduralPerlin}(\vec{scrollPosition}, \text{CloudScale} \times 0.8) $$
    [S2, unofficial]
    Combine densities:
    $$ \text{density} = \text{worleyDensity} \times \text{perlinDensity} \times \text{totalFizzle} $$
    [S2, unofficial]

9.  **Marching Loop:** Loop `i` from `0` to `STEPS - 1`:
    *   **Sample Density:**
        *   If using texture: `density = texture(volumeTex, currentPos).r` [S1, unofficial].
        *   If using procedural: Use the calculated `density` from Step 8 [S2, unofficial].
    *   **Threshold Check:** If `density > 0.01` (texture) or `density > _DensityThreshold` (procedural), proceed; otherwise, skip accumulation for this step [S1, S2, unofficial].
    *   **Accumulate Opacity and Color (Texture Path):**
        *   `color = vec3(density)` [S1, unofficial].
        *   `alpha = density \times \text{DENSITY\_MULTIPLIER} \times \text{stepSize}` [S1, unofficial].
        *   Update accumulation:
            $$ \vec{accum}.rgb += (1.0 - \vec{accum}.a) \times \vec{color} \times \alpha $$
            $$ \vec{accum}.a += (1.0 - \vec{accum}.a) \times \alpha $$
        [S1, unofficial]
    *   **Accumulate Light (Procedural Path):**
        *   Update `densityTotal += density \times \text{stepSize}` [S2, unofficial].
        *   Calculate `sunTransmittance` using a `MarchLight` function [S2, unofficial].
        *   Calculate `cameraTransmittance = \exp(-\text{densityTotal} \times \text{AbsorptionCoefficient})` [S2, unofficial].
        *   Calculate `scatter` using Henyey-Greenstein (HG) phase function [S2, unofficial].
        *   Add to brightness:
            $$ \text{brightness} += \text{density} \times \text{sunTransmittance} \times \text{cameraTransmittance} \times \text{scatter} \times \text{stepSize} \times 0.5 $$
        [S2, unofficial]
    *   **Advance Position:**
        $$ \vec{currentPos} += \vec{deltaStep} $$
        [S1, unofficial]
        If using procedural path, increment `distanceTravelled` by `stepSize` [S2, unofficial].

10. **Final Output:** If `accum.a < 0.001`, discard the fragment [S1, unofficial]. Otherwise, output `outColor = accum` [S1, unofficial].

**Parameters:**
*   `STEPS`: Integer, number of samples per ray. Typical value: 128 [S1, unofficial].
*   `DENSITY_MULTIPLIER`: Float, scales sampled density to alpha. Typical value: 5.0 [S1, unofficial].
*   `_StepSize`: Float, step size for marching (derived from `totalDist / STEPS`) [S2, unofficial].
*   `_StepLimit`: Integer, maximum number of steps [S2, unofficial].
*   `_WindTime`: Float, time variable for scrolling [S2, unofficial].
*   `_CloudScale`: Float, scale factor for procedural noise [S2, unofficial].
*   `_OctaveCount`: Integer, number of FBM octaves for Worley noise [S2, unofficial].
*   `_DensityThreshold`: Float, minimum density for contribution [S2, unofficial].
*   `_AbsorptionCoefficient`: Float, absorption coefficient for transmittance [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The slab method for ray-box intersection must handle cases where the ray is parallel to a slab boundary [S1, unofficial].
*   **Boundaries:** Without `totalFizzle`, volumes may exhibit hard edges at the top and bottom of the bounding box [S2, unofficial].
*   **Degenerate Input:** If `totalDist <= 0.0`, the ray does not intersect the volume, and the fragment should be discarded to avoid undefined behavior [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs
vUVW: vec3
localCameraPos: vec3
boxMin: vec3 = vec3(0.0)
boxMax: vec3 = vec3(1.0)
STEPS: int = 128
DENSITY_MULTIPLIER: float = 5.0
AbsorptionCoefficient: float = 1.0
CloudScale: float = 1.0
WindTime: float = 0.0
OctaveCount: int = 4
DensityThreshold: float = 0.01

# 1. Ray Direction
rayDir = normalize(vUVW - localCameraPos)

# 2. Ray-Box Intersection (Slab Method)
# Simplified for unit box [0,1]^3
invDir = 1.0 / rayDir
tNear = (boxMin - localCameraPos) * invDir
tFar = (boxMax - localCameraPos) * invDir
tMin = max(min(tNear), min(tFar))
tMax = min(max(tNear), max(tFar))
tStart = max(tMin.x, 0.0) # Assuming tMin is vec3 of entry times, taking max component for entry
tEnd = tMax.x # Assuming tMax is vec3 of exit times, taking min component for exit? 
# Note: Standard slab method returns tNear/tFar as scalars. 
# Re-interpreting S1: "vec2(tNear, tFar)". 
# Correct slab logic for axis-aligned box:
t0 = (boxMin - localCameraPos) / rayDir
t1 = (boxMax - localCameraPos) / rayDir
tNear = max(max(t0.x, t0.y), t0.z)
tFar = min(min(t1.x, t1.y), t1.z)

# 3. Validation
totalDist = tFar - tNear
if totalDist <= 0.0:
    discard_fragment()

# 4. Initialization
rayPosition = localCameraPos + rayDir * tNear
densityTotal = 0.0
brightness = 0.0
accum = vec4(0.0)

# 5. Wind Scrolling
scrollPosition = rayPosition + vec3(WindTime, 0.0, 0.0)

# 6. Boundary Fizzle
distToBot = (rayPosition.y - boxMin.y) / (boxMax.y - boxMin.y)
botFizzle = clamp(distToBot * 10.0, 0.0, 1.0)
topFizzle = clamp((1.0 - distToBot) * 10.0, 0.0, 1.0)
totalFizzle = botFizzle * topFizzle

# 7. Step Calculation
stepSize = totalDist / STEPS
currentPos = localCameraPos + rayDir * tNear
deltaStep = rayDir * stepSize

# 8. Procedural Density (Optional Path)
# If using procedural noise instead of texture sampling:
worleyDensity = 0.0
amplitude = 1.0
frequency = 1.0
amplitudeTotal = 0.0
for i in range(OctaveCount):
    worleyDensity += ProceduralWorley(scrollPosition * frequency, 1.0 / CloudScale) * amplitude
    amplitudeTotal += amplitude
    amplitude /= 2.0
    frequency *= 2.0
worleyDensity /= amplitudeTotal

perlinDensity = ProceduralPerlin(scrollPosition, CloudScale * 0.8)
density = worleyDensity * perlinDensity * totalFizzle

# 9. Marching Loop
for i in range(STEPS):
    # Determine density source
    if use_texture:
        density = texture(volumeTex, currentPos).r
        threshold = 0.01
    else:
        threshold = DensityThreshold
    
    if density > threshold:
        if use_texture:
            color = vec3(density)
            alpha = density * DENSITY_MULTIPLIER * stepSize
            accum.rgb += (1.0 - accum.a) * color * alpha
            accum.a += (1.0 - accum.a) * alpha
        else:
            densityTotal += density * stepSize
            # Placeholder for MarchLight and HG scatter calculation as per S2
            # sunTransmittance = MarchLight(...)
            # cameraTransmittance = exp(-densityTotal * AbsorptionCoefficient)
            # scatter = HG(...)
            # brightness += density * sunTransmittance * cameraTransmittance * scatter * stepSize * 0.5
            
    currentPos += deltaStep

# 10. Output
if accum.a < 0.001:
    discard_fragment()
outColor = accum
```

## Subtopics

### Ray Marching Volume Rendering
A broader category of techniques using ray marching for volume rendering. Density Marching is a specific implementation within this category. [Link](1735-ray-marching-volume-rendering.md)

### Ray Evaluation Stepping DVR
Uses variable step sizes based on density evaluation. Density Marching uses fixed steps. [Link](3461-ray-evaluation-stepping-dvr.md)

### Discrete Volume Rendering Integral (VRI)
Focuses on the discrete integration of the volume rendering integral. Density Marching is a numerical approximation of this. [Link](3552-discrete-volume-rendering-integral-vri.md)

### Volumetric Light Shafts (URP Raymarching)
Specifically targets light shafts (god rays) using raymarching. Density Marching targets general volume density. [Link](3873-volumetric-light-shafts-urp-raymarching.md)

## Operating characteristics
*   **Time Complexity:** $O(N \cdot R)$, where $N$ is the number of pixels (fragments) and $R$ is the number of steps (`STEPS`). Since `STEPS` is fixed (e.g., 128), it is linear with respect to pixel count [S1, unofficial].
*   **Memory Complexity:** $O(1)$ per fragment, plus the memory for the 3D texture or procedural noise state [S1, S2, unofficial].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance scales linearly with resolution and step count. Increasing `STEPS` improves quality but increases cost proportionally [S1, unofficial].
*   **Strengths:** Simple to implement; fixed step count ensures predictable performance; procedural noise allows for infinite resolution without texture memory [S1, S2, unofficial].
*   **Limitations:** Fixed step size may lead to aliasing or slow convergence in sparse volumes; no adaptive stepping for efficiency [S1, S2, unofficial].
*   **When to use:** Real-time rendering of cloud-like volumes where performance predictability is key and high-frequency detail is generated via procedural noise [S1, S2, unofficial].
*   **When not to use:** When high accuracy in sparse volumes is required without excessive steps, or when texture memory is limited and procedural noise is too expensive [S1, S2, unofficial].

## Background
Density Marching originates from early direct volume ray casting techniques. The specific implementation described here is found in unofficial repositories for volume rendering tutorials and cloud shaders [S1, S2, unofficial]. It builds on the foundational work of [Fixed Step Ray Marching](3136-fixed-step-ray-marching.md) [S1, unofficial].

## Sources
[S1] vis-magnus-2018, https://github.com/GitThirteen/vis-magnus-2018, fetched 2026-10-26 (unofficial)
[S2] volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
