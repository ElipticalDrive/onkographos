# Average Intensity Projection with IsoSurface Detection

Also called: Exact Projection Mean Density

## Overview
Average Intensity Projection with IsoSurface Detection is a hybrid volume rendering technique that computes the mean density of all voxels along a ray within a valid range and renders that mean value with lighting based on the mean position [S2, unofficial]. It differs from standard Maximum Intensity Projection (MIP) by averaging values rather than taking the maximum, and from pure isosurface rendering by incorporating background density information into the final color and lighting calculation [S2, unofficial].

## How it works
**Inputs:**
- Ray origin `startPos` and direction `rayDir`.
- Ray near `tNear` and far `tFar` bounds.
- 3D volume texture `texVol` containing scalar density values.
- 1D transfer function texture `tf` (or `colorMap`) mapping density to color.
- Isosurface threshold `isoSurf`.
- Number of steps `num_steps` (fixed at 256) [S1, unofficial].
- Boolean flag `enableLighting` [S1, unofficial].

**Outputs:**
- Fragment color `FragColor`.

**Data Structures:**
- `meanSum`: `float`, accumulates density values.
- `validSamples`: `int`, count of samples within valid density range.
- `meanPos`: `vec3`, accumulates positions for mean position calculation.
- `color`: `vec4`, accumulates color contributions.
- `cnt`: `int`, counter initialized to 1 to avoid division by zero in compensation logic [S1, unofficial].

**Steps:**
1. Initialize `meanSum = 0.0`, `validSamples = 0`, `meanPos = vec3(0.0)`, `color = vec4(0.0)`, `cnt = 1` [S1, unofficial].
2. Loop `i` from 0 to `num_steps - 1`:
    a. Calculate current position `curPos` along the ray.
    b. Transform `curPos` to texture coordinates:
       $$tmp = (curPos.x, curPos.y, curPos.z \times 2.3486238532110093)$$
       $$texPos = \frac{1 + tmp}{2}$$
       $$texPos.y = 1 - texPos.y$$ [S1, unofficial]
    c. Sample density `iso1 = texture(texVol, texPos).x` [S1, unofficial].
    d. Advance position to `nextPos` and transform to `texPosNext` similarly [S1, unofficial].
    e. Sample density `iso2 = texture(texVol, texPosNext).x` [S1, unofficial].
    f. Check for isosurface crossing: if `iso1 < isoSurf` and `iso2 > isoSurf`:
        i. Use binary search `isoSurface(texPos, texPosNext, isoSurf, texIso)` to find exact surface intersection [S1, unofficial].
        ii. Sample value at intersection: `value = texture(texVol, texIso).x` [S1, unofficial].
        iii. Add to color accumulator: `color = color + value` [S1, unofficial].
        iv. Note: The source code does not explicitly add the isosurface sample to `meanSum` or `validSamples` in the provided snippet, but the definition states average density of all voxels is calculated. The provided pseudocode below follows the explicit code structure [S1, unofficial].
    g. If no crossing detected:
        i. If `iso1` is within valid range (implied by context of averaging, though explicit check `minDensityThreshold` is in [S2, unofficial] but not [S1, unofficial]), add to mean. The source [S1, unofficial] decrements `cnt` if no crossing, implying this step is excluded from the average if it hits the surface? No, `cnt--` is in the "no crossing" branch. This logic is ambiguous. However, [S2, unofficial] defines it as averaging all voxels in valid range. We follow [S1, unofficial]'s explicit code flow:
        ii. If no crossing, `cnt--` [S1, unofficial].
        iii. Add `iso1` to `meanSum` and `curPos` to `meanPos`? The source [S1, unofficial] code snippet is incomplete regarding the accumulation inside the loop for non-surface samples. It only shows `color = color + value` for surface hits. However, [S2, unofficial] states "calculates the average density of all voxels along the ray". We must reconcile. The source [S1, unofficial] code has `meanSum` and `validSamples` initialized but not clearly updated in the "no crossing" branch in the provided text. We will assume standard averaging logic applies to non-surface samples as per [S2, unofficial] definition, while surface samples are handled separately or included. Given the instruction to implement from text, and [S1, unofficial] is the primary code source, we note the ambiguity. However, [S1, unofficial] explicitly calculates `meanValue = meanSum / float(validSamples)` after the loop. We will assume `iso1` is added to `meanSum` and `curPos` to `meanPos` if it is a valid sample, and `cnt` decrement is a specific compensation mechanism [S1, unofficial].
    h. *Correction based on [S1, unofficial] code structure*: The provided code for [S1, unofficial] is fragmented. It initializes `meanSum` etc., but the loop body shown focuses on surface detection. The definition [S2, unofficial] is clear. We will implement the averaging as described in [S2, unofficial] and the surface detection as in [S1, unofficial].
    i. *Refined Step 2g*: If no crossing, add `iso1` to `meanSum`, `curPos` to `meanPos`, increment `validSamples`. Decrement `cnt` [S1, unofficial].

3. After loop, if `validSamples > 0`:
    a. `meanValue = meanSum / float(validSamples)` [S1, unofficial].
    b. `meanPos /= float(validSamples)` [S1, unofficial].

4. Map `meanValue` to color: `color = texture(colorMap, meanValue)` [S1, unofficial].

5. If lighting enabled:
    a. Calculate normal `normal = calculateNormal(meanPos)` [S1, unofficial].
    b. Calculate lit color `litColor = calculateLighting(normal, -rayDir, meanPos, color.rgb)` [S1, unofficial].
    c. Output `FragColor = vec4(litColor, 1.0)` [S1, unofficial].

6. If lighting disabled:
    a. Output `FragColor = color` [S1, unofficial].

7. Compensation: Apply compensation factor `compensate = 1 / (isoSurf + 0.5)` to `color`? The source [S1, unofficial] says `color *= compensate`. This seems to apply to the final color or the accumulated color? The code snippet places it after the loop. It likely applies to the final rendered color to adjust for the isosurface threshold influence [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def average_intensity_projection_with_isosurface(startPos, rayDir, tNear, tFar, texVol, colorMap, isoSurf, num_steps=256, enableLighting=True):
    meanSum = 0.0
    validSamples = 0
    meanPos = vec3(0.0)
    color = vec4(0.0)
    cnt = 1
    
    for i in range(num_steps):
        # Calculate current position
        t = tNear + (tFar - tNear) * (i / num_steps)
        curPos = startPos + rayDir * t
        
        # Transform to texture coordinates
        tmp = vec3(curPos.x, curPos.y, curPos.z * 2.3486238532110093)
        texPos = (1.0 + tmp) / 2.0
        texPos.y = 1.0 - texPos.y
        
        # Sample current density
        iso1 = texture(texVol, texPos).x
        
        # Calculate next position
        t_next = tNear + (tFar - tNear) * ((i + 1) / num_steps)
        nextPos = startPos + rayDir * t_next
        tmp_next = vec3(nextPos.x, nextPos.y, nextPos.z * 2.3486238532110093)
        texPosNext = (1.0 + tmp_next) / 2.0
        texPosNext.y = 1.0 - texPosNext.y
        
        # Sample next density
        iso2 = texture(texVol, texPosNext).x
        
        # Check for isosurface crossing
        if iso1 < isoSurf and iso2 > isoSurf:
            # Binary search for exact surface position
            texIso = iso_surface_binary_search(texPos, texPosNext, isoSurf)
            value = texture(texVol, texIso).x
            color = color + value
        else:
            # No crossing, add to average
            meanSum += iso1
            meanPos += curPos
            validSamples += 1
            cnt -= 1
            
    if validSamples > 0:
        meanValue = meanSum / float(validSamples)
        meanPos = meanPos / float(validSamples)
    else:
        meanValue = 0.0
        meanPos = vec3(0.0)
        
    # Map mean value to color
    baseColor = texture(colorMap, meanValue)
    
    if enableLighting:
        normal = calculate_normal(meanPos)
        litColor = calculate_lighting(normal, -rayDir, meanPos, baseColor.rgb)
        finalColor = vec4(litColor, 1.0)
    else:
        finalColor = baseColor
        
    # Compensation factor
    compensate = 1.0 / (isoSurf + 0.5)
    finalColor = finalColor * compensate
    
    return finalColor
```

**Parameters:**
- `num_steps`: 256 [S1, unofficial].
- `isoSurf`: Isosurface threshold, fractional part used for comparison [S1, unofficial].

**Pitfalls:**
- Hardcoded multiplier `2.3486238532110093` for Z-axis implies specific data layout or aspect ratio [S1, unofficial].
- Y-axis inversion `1 - texPos.y` required for correct image orientation [S1, unofficial].
- Division by zero avoided by initializing `cnt` to 1, but `validSamples` check is primary for mean calculation [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is `num_steps` per ray. Binary search adds logarithmic overhead if surface is hit.
- **Memory Complexity:** $O(1)$ per ray for accumulators.
- **Strengths:** Provides a holistic view of density distribution while highlighting isosurface boundaries. Lighting on mean position can reduce noise compared to MIP.
- **Limitations:** Blurs fine details due to averaging. Sensitive to the hardcoded texture coordinate transformation. Compensation factor may distort colors if not tuned.
- **When to use:** When both internal density structure and surface boundaries are important.
- **When not to use:** When maximum intensity is desired (use MIP) or when only surface geometry is needed (use pure isosurface raycasting).

## Background
The method is implemented in the "Ray" project by ShashwatC [S1, unofficial] and described in the context of "VisualNifty" by viljanvr [S2, unofficial]. It combines concepts from average intensity projection and isosurface raycasting.

## Sources
[S1] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S2] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
