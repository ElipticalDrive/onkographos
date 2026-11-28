# Basic Volumetric Cone Lighting

Also called: First light ray, Volumetric raymarching loop, Depth-Based Stopping, Scene depth culling

## Overview
Basic Volumetric Cone Lighting is a post-processing technique that renders a volumetric light beam shaped like a cone originating from a light position and pointing in a specific direction, using screen-space raymarching [S1, unofficial]. It solves the problem of rendering visible light shafts (god rays) in real-time by accumulating light density along rays cast from the camera into the scene [S1, unofficial]. It is distinguished from sibling methods like [Shadow Occlusion via Occlusion Camera](16371-shadow-occlusion-via-occlusion-camera.md) by its reliance on a single depth buffer for culling rather than a secondary camera pass, and from [Volume Ray Casting with Marching](14353-volume-ray-casting-with-marching.md) by its specific application to spotlight-shaped volumetric effects rather than general volume density integration [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** `inputColor` (current frame buffer color), `uv` (screen coordinates), `depthBuffer` (scene depth texture), `cameraPosition` (world space), `cameraFar` (max render distance), `lightPosition` (world space), `lightDirection` (normalized world space vector), `coneAngle` (aperture in degrees), `STEP_SIZE` (march increment), `NUM_STEPS` (loop count).
*   **Outputs:** `outputColor` (final pixel color).

**Data Structures:**
*   `depthBuffer`: Texture containing depth values for each pixel.
*   `projectionMatrixInverse`: Inverse of the camera's projection matrix (used for coordinate reconstruction).
*   `viewMatrixInverse`: Inverse of the camera's view matrix (world matrix), used for coordinate reconstruction.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mainImage(inputColor, uv, depthBuffer, cameraFar, projectionMatrixInverse, viewMatrixInverse, cameraPosition, lightPosition, lightDirection, coneAngle, STEP_SIZE, NUM_STEPS):
    # 1. Sample depth and reconstruct world position
    depth = readDepth(depthBuffer, uv)
    worldPosition = getWorldPosition(uv, depth, projectionMatrixInverse, viewMatrixInverse)
    
    # 2. Define ray parameters
    rayOrigin = cameraPosition
    rayDir = normalize(worldPosition - rayOrigin)
    lightPos = lightPosition
    lightDir = normalize(lightDirection)
    
    # 3. Calculate scene depth for culling
    sceneDepth = length(worldPosition - cameraPosition)
    
    # 4. Initialize lighting variables
    coneAngleRad = radians(coneAngle)
    halfConeAngleRad = coneAngleRad * 0.5
    fogAmount = 0.0
    lightIntensity = 1.0
    t = STEP_SIZE
    
    # 5. Raymarching loop
    for i from 0 to NUM_STEPS - 1:
        # Calculate current sample position
        samplePos = rayOrigin + rayDir * t
        
        # Depth-based stopping: break if beyond scene geometry or camera far plane
        if t > sceneDepth or t > cameraFar:
            break
            
        # Cone shape check: calculate angle between sample-to-light vector and light direction
        toSample = normalize(samplePos - lightPos)
        cosAngle = dot(toSample, lightDir)
        
        # Skip samples outside the cone aperture
        if cosAngle < cos(halfConeAngleRad):
            t += STEP_SIZE
            continue
            
        # Attenuation based on distance to light source
        distanceToLight = length(samplePos - lightPos)
        attenuation = exp(-0.05 * distanceToLight)
        
        # Accumulate light
        fogAmount += attenuation * lightIntensity
        
        # Advance ray
        t += STEP_SIZE
        
    # 6. Combine with input color
    outputColor = inputColor + fogAmount
```

**Helper Function (Coordinate Reconstruction):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def getWorldPosition(uv, depth, projectionMatrixInverse, viewMatrixInverse):
    # Convert UV to clip space (-1 to 1)
    clipSpace = vec2(uv.x * 2.0 - 1.0, uv.y * 2.0 - 1.0)
    clipSpace.y = -clipSpace.y # Adjust for different coordinate systems if necessary
    
    # Convert clip space to view space (inverse projection)
    viewSpace = vec4(clipSpace, -1.0, 1.0) # w=1 for projection inverse
    viewSpace = projectionMatrixInverse * viewSpace
    viewSpace = vec4(viewSpace.xyz / viewSpace.w, viewSpace.w)
    
    # Convert view space to world space (inverse view)
    worldSpace = viewMatrixInverse * vec4(viewSpace.xyz, 1.0)
    return worldSpace.xyz
```

**Parameters:**
*   `cameraFar`: Depth limit beyond which nothing is visible or rendered. Typical values depend on scene scale (e.g., 500 units) [S1, unofficial].
*   `coneAngle`: Spotlight aperture width in degrees. Typical values range from 10 to 60 degrees [S1, unofficial].
*   `STEP_SIZE`: Step size for raymarching. Not explicitly given in sources; must be tuned for performance/quality trade-off [S1, unofficial].
*   `NUM_STEPS`: Number of steps in the raymarching loop. Not explicitly given in sources; depends on `STEP_SIZE` and `cameraFar` [S1, unofficial].
*   `0.05`: Exponential decay constant for light attenuation. This value is hardcoded in the source example [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Through-wall leakage:** Without depth-based stopping, the light ray continues marching beyond visible geometry, causing light to appear through walls and objects, breaking realism and wasting performance [S1, unofficial].
*   **Precision:** Reconstruction of world position from depth buffer requires accurate inverse matrices. Incorrect matrix usage leads to misaligned light rays [S1, unofficial].
*   **Cone Angle Calculation:** The cosine check `cosAngle < cos(halfConeAngleRad)` assumes the light direction points *away* from the source in the direction of the beam. If `lightDirection` is defined differently, the dot product logic may need inversion [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(NUM\_STEPS)$ per pixel. The loop runs a fixed number of steps, potentially breaking early due to depth culling.
*   **Memory Complexity:** $O(1)$ auxiliary memory per pixel, plus the depth buffer texture.
*   **Performance:** The method is relatively lightweight compared to full volumetric integration but can be expensive if `NUM_STEPS` is high or if depth culling is ineffective (e.g., in open scenes). The exponential attenuation calculation (`exp`) is computationally moderate.
*   **Strengths:** Simple to implement in post-processing; provides convincing volumetric light shafts; depth culling prevents visual artifacts through geometry.
*   **Limitations:** Only works for post-processing (cannot affect objects behind the camera or in front of the near plane easily); relies on a single light source direction/position per effect instance; no scattering or multiple bounces; quality depends heavily on `STEP_SIZE` and `NUM_STEPS` [S1, unofficial].
*   **When to use:** Real-time rendering of god rays or spotlight beams in post-processing.
*   **When not to use:** When physically accurate volumetric scattering is required, or when lights need to interact with complex volumetric media (fog, smoke) defined in world space rather than screen space.

## Background
The technique was popularized in the context of real-time web graphics and post-processing frameworks like Three.js. It is described in "On Shaping Light: Real-Time Volumetric Lighting with Post-Processing and Raymarching" by Maxime Heckel, which details the implementation of volumetric lighting effects using raymarching in fragment shaders [S1, unofficial]. The method builds on the principles of screen-space raymarching and depth buffer reconstruction.

## Sources
[S1] On Shaping Light: Real-Time Volumetric Lighting with Post-Processing ..., https://blog.maximeheckel.com/posts/shaping-light-volumetric-lighting-with-post-processing-and-raymarching/, fetched 2026-10-28 (unofficial)
