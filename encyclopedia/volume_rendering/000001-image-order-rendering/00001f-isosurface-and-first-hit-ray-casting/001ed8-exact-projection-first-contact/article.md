# Exact Projection First Contact

Also called: FHC, First Hit, First-Hit Compositing
## Overview
Exact Projection First Contact is a volume rendering technique that identifies the first voxel along a ray whose density value falls within a specified valid range, then applies shading and occlusion to that specific point [S1, S2, unofficial]. It differs from standard volume ray casting by discarding all subsequent samples after the first valid hit, effectively treating the volume as an opaque surface defined by the threshold [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin (`startPos`), ray direction (`rayDir`), near plane (`tNear`), far plane (`tFar`), volume texture (`volumeTexture`), color map texture (`colorMap`), minimum density threshold (`minDensityThreshold`), maximum density threshold (`maxDensityThreshold`), lighting flags (`enableLighting`, `enableAO`, `enableShadow`), and volume dimensions (`volumeDimensions`) [S1, unofficial].
*   **Outputs:** Final fragment color (`FragColor`).

**Data Structures:**
*   `volumeTexture`: 3D texture storing scalar density values [S1, unofficial].
*   `colorMap`: 1D or 2D texture mapping density values to RGBA colors [S1, unofficial].
*   `firstContactVal`: Float, stores the density value of the first hit. Initialized to `0.0` [S1, unofficial].
*   `firstContactPos`: Vec3, stores the 3D position of the first hit. Initialized to `startPos` [S1, unofficial].
*   `samplePos`: Vec3, current sampling position.
*   `t`: Float, current ray parameter.
*   `iterations`: Integer, loop counter.
*   `MAX_ITERATIONS`: Integer, safety limit (e.g., 2000) [S1, unofficial].

**Steps:**
1.  Normalize the ray direction: `rayDir = normalize(rayDir)` [S3, unofficial].
2.  Initialize `firstContactVal = 0.0` and `firstContactPos = startPos` [S1, unofficial].
3.  Set `t = tNear` and `iterations = 0` [S1, unofficial].
4.  Loop while `t < tFar` and `iterations < MAX_ITERATIONS`:
    a.  Calculate `samplePos = startPos + rayDir * t` [S1, unofficial].
    b.  Check bounds: If `samplePos` is outside the unit cube `[0,1]^3`, compute step size `dt`, advance `t += dt`, increment `iterations`, and continue [S1, unofficial].
    c.  Sample density: `val = texture(volumeTexture, samplePos).r` [S1, unofficial].
    d.  Check range: `isInRange = (val >= minDensityThreshold && val <= maxDensityThreshold)` [S1, unofficial].
    e.  If `isInRange` is true: set `firstContactVal = val`, `firstContactPos = samplePos`, and break the loop [S1, unofficial].
    f.  Compute step size `dt` using voxel traversal logic, advance `t += dt`, and increment `iterations` [S1, unofficial].
5.  If `firstContactVal <= 0.0f`, output `FragColor = vec4(0.0f)` and return [S1, unofficial].
6.  Map value to color: `baseColor = texture(colorMap, firstContactVal)` [S1, unofficial].
7.  Calculate lighting: If `enableLighting` is true, compute the surface normal via gradient estimation, then compute `shadedColor` using a local lighting model; else `shadedColor = baseColor.rgb` [S1, unofficial].
8.  Calculate ambient occlusion: `occlusion = enableAO ? calculateOcclusion(firstContactPos) : 1.0f` [S1, unofficial].
9.  Calculate shadow: `shadow = enableShadow ? calculateShadow(firstContactPos) : 1.0f` [S1, unofficial].
10. Output final color: `FragColor = vec4(shadedColor * occlusion * shadow, 1.0f)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exact_projection_first_contact(startPos, rayDir, tNear, tFar, volumeTexture, colorMap, minDensityThreshold, maxDensityThreshold, enableLighting, enableAO, enableShadow, volumeDimensions):
    # Normalize ray direction [S3, unofficial]
    rayDir = rayDir / length(rayDir)
    
    firstContactVal = 0.0
    firstContactPos = startPos
    t = tNear
    MAX_ITERATIONS = 2000
    iterations = 0
    
    # Helper to compute step size dt for voxel traversal [S1, unofficial]
    def compute_dt(pos, rdir, vdim):
        voxelPos = pos * vdim
        dt = float('inf')
        for axis in range(3):
            if abs(rdir[axis]) > 1e-4:
                nextVoxel = floor(voxelPos[axis]) + 1.0 if rdir[axis] > 0 else floor(voxelPos[axis])
                dtAxis = (nextVoxel - voxelPos[axis]) / (rdir[axis] * vdim[axis])
                if dtAxis > 0:
                    dt = min(dt, dtAxis)
        return max(dt, 1e-4)

    while t < tFar and iterations < MAX_ITERATIONS:
        samplePos = startPos + rayDir * t
        
        # Check if inside unit cube [0,1]^3
        if samplePos.x < 0.0 or samplePos.y < 0.0 or samplePos.z < 0.0 or \
           samplePos.x > 1.0 or samplePos.y > 1.0 or samplePos.z > 1.0:
            dt = compute_dt(samplePos, rayDir, volumeDimensions)
            t += dt
            iterations += 1
            continue
            
        val = volumeTexture[samplePos] # [G] texture lookup
        isInRange = (val >= minDensityThreshold and val <= maxDensityThreshold)
        
        if isInRange:
            firstContactVal = val
            firstContactPos = samplePos
            break
            
        dt = compute_dt(samplePos, rayDir, volumeDimensions)
        t += dt
        iterations += 1
        
    if firstContactVal <= 0.0:
        return vec4(0.0, 0.0, 0.0, 1.0)
        
    baseColor = colorMap[firstContactVal] # [G] color map lookup
        
    if enableLighting:
        # Compute normal via central finite differences [G]
        eps = vec3(1.0 / volumeDimensions[0], 1.0 / volumeDimensions[1], 1.0 / volumeDimensions[2])
        p1 = firstContactPos + vec3(eps[0], 0, 0)
        p2 = firstContactPos - vec3(eps[0], 0, 0)
        p3 = firstContactPos + vec3(0, eps[1], 0)
        p4 = firstContactPos - vec3(0, eps[1], 0)
        p5 = firstContactPos + vec3(0, 0, eps[2])
        p6 = firstContactPos - vec3(0, 0, eps[2])
        normal = normalize(vec3(
            volumeTexture[p1] - volumeTexture[p2],
            volumeTexture[p3] - volumeTexture[p4],
            volumeTexture[p5] - volumeTexture[p6]
        ))
        # Simple Phong lighting [G]
        viewDir = normalize(-rayDir)
        lightDir = normalize(vec3(0.5, 1.0, 0.5))
        diff = max(dot(normal, lightDir), 0.0)
        spec = pow(max(dot(reflect(-lightDir, normal), viewDir), 0.0), 32.0)
        shadedColor = baseColor.rgb * (0.5 + 0.5 * diff) + vec3(0.3) * spec
    else:
        shadedColor = baseColor.rgb
        
    occlusion = 1.0 # [G] calculateOcclusion returns 1.0 if not specified
    shadow = 1.0 # [G] calculateShadow returns 1.0 if not specified
        
    return vec4(shadedColor * occlusion * shadow, 1.0)
```

**Parameters:**
*   `minDensityThreshold`, `maxDensityThreshold`: Define the valid density range. Typical values depend on the dataset normalization.
*   `MAX_ITERATIONS`: 2000 [S1, unofficial].
*   `stepSize`/`dt`: Determined by voxel traversal or fixed step.

**Edge Cases and Pitfalls:**
*   **No Hit:** If no voxel in the ray path falls within the density range, the function outputs black [S1, unofficial].
*   **Precision:** Floating-point precision in `t` advancement can cause missed voxels or double sampling. Voxel traversal logic mitigates this [S1, unofficial].
*   **Boundaries:** Rays starting outside the volume or exiting early must be handled by the bounds check [S1, unofficial].
*   **Ray Direction:** The ray direction may not be normalized after interpolation from the vertex shader; it must be explicitly normalized before use [S3, unofficial].

## Operating characteristics
*   **Time Complexity:** O(N) where N is the number of voxels traversed until the first hit or exit [G].
*   **Memory Complexity:** O(1) auxiliary memory, plus storage for the volume and color map textures [G].
*   **Performance:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to Use:** Not given in the sources.
*   **When Not to Use:** Not given in the sources.

## Background
Exact Projection First Contact is a fundamental technique in volume rendering, often used as a baseline for more complex methods. It is closely related to the concept of "first hit" in ray tracing. The specific implementation details provided in the sources are from unofficial repositories [S1, S2, unofficial].

## Sources
[S1] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S2] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S3] Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, https://github.com/Masdaofdisasda/Vis1_Direct_Volume_Rendering_2022W, fetched 2026-10-26 (unofficial)
