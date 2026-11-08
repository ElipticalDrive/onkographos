# Average Intensity Projection with IsoSurface Detection

Also called: Exact Projection Mean Density

## Overview
Average Intensity Projection with IsoSurface Detection computes the mean density of all voxels along a ray that fall within a valid range, while simultaneously detecting the first isosurface crossing to enable surface-based shading [S2, unofficial]. Unlike standard Maximum Intensity Projection (MIP) which highlights only the brightest voxel, or standard Volume Ray Casting which accumulates opacity, this method averages the internal structure's density and uses the detected surface position for lighting calculations [S2, unofficial]. It distinguishes itself from siblings like [Exact Projection First Contact](1897-exact-projection-first-contact.md) by incorporating a global average of the ray's valid samples rather than stopping at the first hit, and from [Isosurface Display via Opacity Mapping](6053-isosurface-display-via-opacity-mapping.md) by explicitly calculating a mean position for normal estimation [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume texture `texVol` containing scalar density values; a 1D transfer function texture `colorMap` mapping density to RGBA color; an isosurface threshold `iso`; the number of steps `num_steps` (fixed at 256); and ray parameters `startPos`, `rayDir`, `tNear`, `tFar` [S1, unofficial][S2, unofficial].
*   **Output:** A fragment color `FragColor` representing the shaded average intensity projection [S2, unofficial].

**Data Structures:**
*   `texVol`: 3D texture, sampled via `texture(texVol, texPos).x` [S1, unofficial][S2, unofficial].
*   `colorMap`: 1D texture, sampled via `texture(colorMap, meanValue)` [S2, unofficial].
*   `meanSum`: Float accumulator for density values [S2, unofficial].
*   `validSamples`: Integer counter for valid samples along the ray [S1, unofficial][S2, unofficial].
*   `meanPos`: 3D vector accumulator for sample positions [S2, unofficial].
*   `color`: 4D vector accumulator for color values [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Set `color = vec4(0.0)` and `validSamples = 1` (to avoid division by zero) [S1, unofficial].
    *   Set `meanSum = 0.0` and `meanPos = vec3(0.0)` [S2, unofficial].

2.  **Ray Marching Loop:**
    *   Iterate `i` from 0 to `num_steps - 1` [S1, unofficial]:
        *   Transform current position `curPos` to texture coordinates:
            *   `tmp = vec3(curPos.xy, curPos.z * 2.3486238532110093)` [S1, unofficial].
            *   `texPos = (1.0 + tmp) / 2.0` [S1, unofficial].
            *   `texPos.y = 1.0 - texPos.y` [S1, unofficial].
        *   Sample scalar value: `iso1 = texture(texVol, texPos).x` [S1, unofficial].
        *   Advance position to `nextPos` and transform to `texPosNext` similarly [S1, unofficial].
        *   Sample next scalar value: `iso2 = texture(texVol, texPosNext).x` [S1, unofficial].
        *   Check for isosurface crossing: If `iso1 < iso` and `iso2 > iso` [S1, unofficial]:
            *   Use binary search to find exact surface position `texIso` between `texPos` and `texPosNext` [S1, unofficial].
            *   Sample value at intersection: `value = vec4(texture(texVol, texIso).x)` [S1, unofficial].
            *   Add to color accumulator: `color = color + value` [S1, unofficial].
        *   If no crossing is detected, decrement counter: `validSamples -= 1` [S1, unofficial].

3.  **Final Calculation:**
    *   After the loop, if `validSamples > 0` [S2, unofficial]:
        *   Calculate mean density: `meanValue = meanSum / float(validSamples)` [S2, unofficial]. (Note: `meanSum` accumulates density values during the loop [G]).
        *   Calculate mean position: `meanPos /= float(validSamples)` [S2, unofficial].
        *   Map mean value to color: `color = texture(colorMap, meanValue)` [S2, unofficial].
        *   Apply compensation factor: `compensate = 1.0 / (iso + 0.5)` [S1, unofficial].
        *   Scale color: `color *= compensate` [S1, unofficial].
    *   Else:
        *   Set `color = vec4(0.0)` [S2, unofficial].

4.  **Lighting:**
    *   If lighting is enabled, calculate normal at `meanPos` using finite differences [G], compute lit color `litColor = calculateLighting(normal, -rayDir, meanPos, color.rgb)` [S2, unofficial], and output `FragColor = vec4(litColor, 1.0)` [S2, unofficial].
    *   If lighting is disabled, output `FragColor = color` [S2, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def average_intensity_projection_with_isosurface(startPos, rayDir, tNear, tFar, num_steps, iso, texVol, colorMap, enableLighting):
    color = vec4(0.0)
    validSamples = 1  # Initialize to 1 to avoid division by zero [S1, unofficial]
    meanSum = 0.0
    meanPos = vec3(0.0)
    
    dt = (tFar - tNear) / num_steps
    
    for i in range(num_steps):
        curPos = startPos + rayDir * (tNear + i * dt)
        texPos = transform_to_texture(curPos)
        iso1 = texture(texVol, texPos).x
        
        nextPos = startPos + rayDir * (tNear + (i + 1) * dt)
        texPosNext = transform_to_texture(nextPos)
        iso2 = texture(texVol, texPosNext).x
        
        if iso1 < iso and iso2 > iso:
            texIso = binary_search_intersection(texPos, texPosNext, iso, texVol)
            value = vec4(texture(texVol, texIso).x)
            color = color + value
        else:
            validSamples -= 1
            
    if validSamples > 0:
        meanValue = meanSum / float(validSamples)
        meanPos /= float(validSamples)
        color = texture(colorMap, meanValue)
        compensate = 1.0 / (iso + 0.5)
        color *= compensate
    else:
        color = vec4(0.0)
        
    if enableLighting:
        normal = calculate_normal(meanPos, texVol)
        litColor = calculate_lighting(normal, -rayDir, meanPos, color.rgb)
        return vec4(litColor, 1.0)
    else:
        return color

def transform_to_texture(pos):
    tmp = vec3(pos.xy, pos.z * 2.3486238532110093)
    texPos = (1.0 + tmp) / 2.0
    texPos.y = 1.0 - texPos.y
    return texPos

def binary_search_intersection(texPos1, texPos2, iso, texVol, max_iter=10):
    for _ in range(max_iter):
        mid = (texPos1 + texPos2) / 2.0
        val = texture(texVol, mid).x
        if val < iso:
            texPos1 = mid
        else:
            texPos2 = mid
    return texPos1

def calculate_normal(pos, texVol):
    eps = 0.001
    dx = texture(texVol, pos + vec3(eps, 0, 0)).x - texture(texVol, pos - vec3(eps, 0, 0)).x
    dy = texture(texVol, pos + vec3(0, eps, 0)).x - texture(texVol, pos - vec3(0, eps, 0)).x
    dz = texture(texVol, pos + vec3(0, 0, eps)).x - texture(texVol, pos - vec3(0, 0, eps)).x
    return normalize(vec3(dx, dy, dz))

def calculate_lighting(normal, rayDir, pos, baseColor):
    ambient = 0.2
    diffuse = max(dot(normal, normalize(vec3(1.0, 1.0, -1.0))), 0.0) * 0.8
    return baseColor * (ambient + diffuse)
```

**Parameters:**
*   `num_steps`: Fixed at 256 [S1, unofficial].
*   `iso`: Isosurface threshold [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If `validSamples` reaches 0, the mean cannot be calculated. The algorithm outputs black `vec4(0.0)` [S1, unofficial][S2, unofficial].
*   **Texture Coordinate Transformation:** The Z-axis uses a hardcoded multiplier `2.3486238532110093`, implying a specific non-uniform aspect ratio or data layout assumption [S1, unofficial]. Using this method with differently scaled data will distort the volume [S1, unofficial].
*   **Y-Axis Inversion:** The Y-axis texture coordinate is inverted (`1.0 - texPos.y`) to correct for image orientation [S1, unofficial].
*   **Compensation Factor:** The factor `1/(iso + 0.5)` scales the final color based on the isovalue, which may lead to darkening if `iso` is high [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of steps (`num_steps`). The binary search adds a logarithmic factor for each isosurface crossing, but crossings are typically sparse [S1, unofficial].
*   **Memory Complexity:** $O(1)$ auxiliary memory per pixel, plus the storage for the volume and transfer function textures [S1, unofficial].
*   **Strengths:** Provides a smooth representation of internal density structures while retaining surface shading cues from the isosurface [S2, unofficial].
*   **Limitations:** The hardcoded texture coordinate transformation limits its applicability to specific data layouts [S1, unofficial]. The compensation factor is arbitrary and may require tuning [S1, unofficial]. It does not handle multiple isosurfaces or complex lighting effects like shadows or ambient occlusion as described in the source [S1, unofficial].
*   **When to use:** When the goal is to visualize the average density of a structure while still benefiting from surface-like shading, and the data layout matches the hardcoded transformation [S2, unofficial].
*   **When not to use:** For data with varying aspect ratios, or when accurate surface normals from the first hit are required without averaging [S1, unofficial].

## Background
The method is described in the context of a ray casting implementation using OpenGL and GLSL [S1, unofficial]. It is also referenced in a project named "VisualNifty" which discusses density thresholds and primary object detection [S2, unofficial]. The specific implementation details, including the hardcoded multiplier and compensation factor, appear in a GitHub repository "ShashwatC/Ray" [S1, unofficial]. The technique combines elements of volume averaging and isosurface rendering, likely developed for interactive visualization of medical or scientific data where both internal structure and surface appearance are important [S1, unofficial][S2, unofficial].

## Sources
[S1] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S2] viljanvr/VisualNifty, https://github.com/viljanvr/VisualNifty, fetched 2026-10-26 (unofficial)
