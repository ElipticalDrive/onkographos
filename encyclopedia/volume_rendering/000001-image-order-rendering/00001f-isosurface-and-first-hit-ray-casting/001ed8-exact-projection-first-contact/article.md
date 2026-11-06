# Exact Projection First Contact

Also called: First Hit
## Overview
Exact Projection First Contact is a volume rendering technique that identifies the first voxel along a ray whose density value falls within a specified valid range, then applies shading and occlusion to that specific point [S1, S2, unofficial]. It differs from standard volume ray casting by discarding all subsequent samples after the first valid hit, effectively treating the volume as an opaque surface defined by the threshold [S1, S2, unofficial]. This method is distinct from [Isosurface and first-hit ray casting](20-isosurface-and-first-hit-ray-casting.md) in its explicit projection-based traversal logic rather than implicit surface extraction, and from [Volumetric Isosurface Rendering with Intersection Refinement](958-volumetric-isosurface-rendering-with-intersection-refinement.md) by skipping refinement steps in favor of direct first-hit detection [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin (`startPos`), ray direction (`rayDir`), near plane (`tNear`), far plane (`tFar`), volume texture (`volumeTexture`), color map texture (`colorMap`), minimum density threshold (`minDensityThreshold`), maximum density threshold (`maxDensityThreshold`), lighting flags (`enableLighting`, `enableAO`, `enableShadow`), and shading mode.
*   **Outputs:** Final fragment color (`FragColor`).

**Data Structures:**
*   `volumeTexture`: 3D texture storing scalar density values. Access via `texture(volumeTexture, samplePos).r`.
*   `colorMap`: 1D or 2D texture mapping density values to RGBA colors. Access via `texture(colorMap, normalizedVal)`.
*   `firstContactVal`: Float, stores the density value of the first hit. Initialized to `0.0` [S1, unofficial].
*   `firstContactPos`: Vec3, stores the 3D position of the first hit. Initialized to `startPos` [S1, unofficial].
*   `samplePos`: Vec3, current sampling position.
*   `t`: Float, current ray parameter.
*   `iterations`: Integer, loop counter.
*   `MAX_ITERATIONS`: Integer, safety limit (e.g., 2000) [S1, unofficial].

**Steps:**
1.  Initialize `firstContactVal = 0.0` and `firstContactPos = startPos` [S1, unofficial].
2.  Set `t = tNear` [S1, unofficial].
3.  Loop while `t < tFar` and `iterations < MAX_ITERATIONS`:
    a.  Calculate `samplePos = startPos + rayDir * t` [S1, unofficial].
    b.  Check bounds: If `any(lessThan(samplePos, vec3(0.0)))` or `any(greaterThan(samplePos, vec3(1.0)))`, increment `t` by `stepSize` (or `dt`), increment `iterations`, and continue [S1, unofficial].
    c.  Sample density: `val = texture(volumeTexture, samplePos).r` [S1, unofficial].
    d.  Check range: `isInRange = (val >= minDensityThreshold && val <= maxDensityThreshold)` [S1, unofficial].
    e.  If `isInRange` is true:
        i.  Set `firstContactVal = val` [S1, unofficial].
        ii. Set `firstContactPos = samplePos` [S1, unofficial].
        iii. Break the loop [S1, unofficial].
    f.  Advance `t`: In [S1, unofficial], `t` is advanced by `dt` (calculated via voxel traversal logic) or `stepSize`. In [S2, unofficial], `t` is advanced by `tIncr * i` in a fixed-step loop. The robust implementation uses voxel traversal [S1, unofficial].
    g.  Increment `iterations` [S1, unofficial].
4.  If `firstContactVal <= 0.0f`, output `FragColor = vec4(0.0f)` and return [S1, unofficial].
5.  Map value to color: `baseColor = texture(colorMap, firstContactVal)` [S1, unofficial].
6.  Calculate lighting:
    a.  If `enableLighting` is true:
        i.  Calculate normal: `normal = calculateNormal(firstContactPos)` [S1, unofficial].
        ii. Calculate shaded color: `shadedColor = calculateLighting(normal, -rayDir, firstContactPos, baseColor.rgb)` [S1, unofficial].
    b.  Else, `shadedColor = baseColor.rgb` [S1, unofficial].
7.  Calculate ambient occlusion: `occlusion = enableAO ? calculateOcclusion(firstContactPos) : 1.0f` [S1, unofficial].
8.  Calculate shadow: `shadow = enableShadow ? calculateShadow(firstContactPos) : 1.0f` [S1, unofficial].
9.  Output final color: `FragColor = vec4(shadedColor * occlusion * shadow, 1.0f)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def exact_projection_first_contact(startPos, rayDir, tNear, tFar, volumeTexture, colorMap, minDensityThreshold, maxDensityThreshold, enableLighting, enableAO, enableShadow):
    firstContactVal = 0.0
    firstContactPos = startPos
    t = tNear
    MAX_ITERATIONS = 2000
    iterations = 0
    
    # Helper for voxel traversal step size calculation (from S1)
    def get_dt(samplePos, rayDir, volumeDimensions):
        voxelPos = samplePos * volumeDimensions
        dtVec = vec3(1.0, 1.0, 1.0)
        if abs(rayDir.x) > 0.0001:
            nextVoxelX = floor(voxelPos.x) + 1.0 if rayDir.x > 0 else floor(voxelPos.x)
            dtVec.x = (nextVoxelX - voxelPos.x) / (rayDir.x * volumeDimensions.x)
        if abs(rayDir.y) > 0.0001:
            nextVoxelY = floor(voxelPos.y) + 1.0 if rayDir.y > 0 else floor(voxelPos.y)
            dtVec.y = (nextVoxelY - voxelPos.y) / (rayDir.y * volumeDimensions.y)
        if abs(rayDir.z) > 0.0001:
            nextVoxelZ = floor(voxelPos.z) + 1.0 if rayDir.z > 0 else floor(voxelPos.z)
            dtVec.z = (nextVoxelZ - voxelPos.z) / (rayDir.z * volumeDimensions.z)
        dt = min(min(dtVec.x, dtVec.y), dtVec.z)
        return max(dt, 0.0001)

    while t < tFar and iterations < MAX_ITERATIONS:
        samplePos = startPos + rayDir * t
        
        # Check if inside volume bounds [0,1]
        if any(samplePos < 0.0) or any(samplePos > 1.0):
            # Use voxel traversal for dt if available, else fixed step
            # Assuming volumeDimensions is available for traversal
            dt = get_dt(samplePos, rayDir, volumeDimensions) # [G] volumeDimensions needed for traversal
            t += dt
            iterations += 1
            continue
            
        val = texture(volumeTexture, samplePos).r
        isInRange = (val >= minDensityThreshold and val <= maxDensityThreshold)
        
        if isInRange:
            firstContactVal = val
            firstContactPos = samplePos
            break
            
        dt = get_dt(samplePos, rayDir, volumeDimensions) # [G]
        t += dt
        iterations += 1
        
    if firstContactVal <= 0.0:
        return vec4(0.0, 0.0, 0.0, 1.0)
        
    baseColor = texture(colorMap, firstContactVal)
    
    if enableLighting:
        normal = calculate_normal(firstContactPos) # [G] Implementation depends on gradient estimation
        shadedColor = calculate_lighting(normal, -rayDir, firstContactPos, baseColor.rgb)
    else:
        shadedColor = baseColor.rgb
        
    occlusion = calculate_occlusion(firstContactPos) if enableAO else 1.0
    shadow = calculate_shadow(firstContactPos) if enableShadow else 1.0
    
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

## Subtopics

### First Hit
A general term for any method that stops at the first surface intersection. Exact Projection First Contact is a specific implementation using density thresholds [S1, S2, unofficial].

### Voxel Traversal
The method of advancing `t` by voxel boundaries rather than fixed steps, improving efficiency and accuracy [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** O(N) where N is the number of voxels traversed until the first hit or exit. In the worst case (no hit or hit at the end), it traverses the entire ray length.
*   **Memory Complexity:** O(1) auxiliary memory, plus storage for the volume and color map textures.
*   **Performance:** Fast for sparse volumes or when the isosurface is near the entry point. Slower than MIP if the first hit is deep, but faster than full volume integration.
*   **Strengths:** Simple implementation, fast rendering for opaque surfaces, good for interactive visualization of dense structures.
*   **Limitations:** Loses internal volume information, sensitive to threshold selection, can produce aliasing if step size is too large (mitigated by voxel traversal).
*   **When to Use:** When only the outer surface of a volume is of interest, and transparency/translucency is not required.
*   **When Not to Use:** When internal structures need to be visible, or when the volume is highly transparent.

## Background
Exact Projection First Contact is a fundamental technique in volume rendering, often used as a baseline for more complex methods. It is closely related to the concept of "first hit" in ray tracing. The specific implementation details provided in the sources are from unofficial repositories [S1, S2, unofficial].

## Sources
[S1] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
[S2] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
