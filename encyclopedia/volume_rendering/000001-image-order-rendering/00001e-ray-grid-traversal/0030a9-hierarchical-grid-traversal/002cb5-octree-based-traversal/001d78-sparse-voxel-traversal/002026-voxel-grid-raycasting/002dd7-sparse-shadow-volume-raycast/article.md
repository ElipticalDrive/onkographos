# Sparse Shadow Volume Raycast

Also called: raycastShadowVolumeSparse

## Overview
A hierarchical raycasting algorithm that traverses a sparse shadow volume texture to find the first intersection with a shadow-casting volume [S1, unofficial]. It differs from fixed-step hierarchical variants by dynamically adjusting its traversal step size and level of detail (LOD) based on occupancy bits, allowing it to skip empty space efficiently in sparse representations [S1, unofficial].

## How it works
**Inputs:**
*   `origin`: Ray origin in world space.
*   `dir`: Ray direction (normalized).
*   `dist`: Maximum ray distance.
*   `uVolOffset`: Offset vector for the shadow volume texture coordinates.
*   `uVolTexelSize`: Physical size of a texel in the shadow volume.
*   `uVolResolution`: Resolution of the shadow volume texture (vec3).
*   `uShadowVolume`: The sparse shadow volume texture (likely a sparse voxel octree or similar structure).

**Outputs:**
*   `d`: Distance to the first occupied voxel. Returns `dist` if no intersection is found within the limit.

**Data Structures:**
*   `uShadowVolume`: A texture where each texel contains occupancy information. The algorithm samples this texture at different LODs. The exact internal structure (e.g., sparse voxel octree encoding) is abstracted by the `textureLod` call, but the algorithm relies on the `x` component of the sampled value being non-zero to indicate occupancy [S1, unofficial].

**Steps:**

1.  **Initialization:**
    *   Translate origin: `origin = origin + uVolOffset`.
    *   Compute step size: `step = uVolTexelSize`.
    *   Compute step direction in texture space: `stepDir = dir * (1.0 / uVolResolution)`.
    *   Compute initial position in texture space: `pos = origin * (1.0 / uVolTexelSize) * (1.0 / uVolResolution)`.
    *   Halve step parameters: `stepDir = stepDir * 0.5`, `step = step * 0.5`.
    *   Initialize LOD: `lod = -1`.
    *   Initialize distance: `d = 0.0`.

2.  **Traversal Loop:**
    *   While `d < dist`:
        *   Sample texture: `c = textureLod(uShadowVolume, pos, lod).x`.
        *   **If `lod == -1` (Coarse Search):**
            *   If `c != 0`:
                *   Calculate bit index from fractional parts of `pos`:
                    *   `bit = (fract(pos.x * uVolResolution.x) > 0.5 ? 1 : 0) + (fract(pos.y * uVolResolution.y) > 0.5 ? 2 : 0) + (fract(pos.z * uVolResolution.z) > 0.5 ? 4 : 0)`.
                *   Create mask: `mask = 1 << bit`.
                *   If `(mask & c) != 0`: Return `d`.
                *   Else: Advance `pos += stepDir`, `d += step`.
            *   If `c == 0`:
                *   Increment `lod`.
                *   Double `stepDir` and `step`.
                *   Advance `pos += stepDir`, `d += step`.
        *   **If `lod != -1` (Refinement):**
            *   If `c != 0`:
                *   Halve `stepDir` and `step`.
                *   Decrement `lod`.
                *   Backtrack: `pos -= stepDir`, `d -= step`.
            *   If `c == 0`:
                *   If `lod < 2`:
                    *   Sample `c = textureLod(uShadowVolume, pos, lod + 1).x`.
                    *   If `c == 0`:
                        *   Increment `lod`.
                        *   Double `stepDir` and `step`.
                        *   Advance `pos += stepDir`, `d += step`.
                *   Else (`lod == 2`):
                    *   Do not go deeper. Rely on coarse search or distance limit. [S1, unofficial]

**Parameters:**
*   `uVolResolution`: vec3, resolution of the shadow volume texture.
*   `uVolTexelSize`: float, physical size of a texel.

**Edge Cases and Pitfalls:**
*   The algorithm assumes a maximum LOD of 2. If `lod` is already 2 and the sample is empty, it does not attempt to go deeper, relying on the coarse search or returning the distance limit [S1, unofficial].
*   Precision issues may arise when calculating bit indices from fractional parts of `pos` if `pos` values are very large or small.
*   The algorithm assumes the shadow volume texture supports `textureLod` with varying LOD levels.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def raycast_shadow_volume_sparse(origin, dir, dist, uVolOffset, uVolTexelSize, uVolResolution, uShadowVolume):
    origin = origin + uVolOffset
    step = uVolTexelSize
    stepDir = dir * (1.0 / uVolResolution)
    pos = origin * (1.0 / uVolTexelSize) * (1.0 / uVolResolution)
    stepDir = stepDir * 0.5
    step = step * 0.5
    lod = -1
    d = 0.0

    while d < dist:
        c = textureLod(uShadowVolume, pos, lod).x
        if lod == -1:
            if c != 0:
                bit = (1 if fract(pos.x * uVolResolution.x) > 0.5 else 0) + \
                      (2 if fract(pos.y * uVolResolution.y) > 0.5 else 0) + \
                      (4 if fract(pos.z * uVolResolution.z) > 0.5 else 0)
                mask = 1 << bit
                if (mask & c) != 0:
                    return d
                else:
                    pos = pos + stepDir
                    d = d + step
            else:
                lod = lod + 1
                stepDir = stepDir * 2.0
                step = step * 2.0
                pos = pos + stepDir
                d = d + step
        else:
            if c != 0:
                stepDir = stepDir * 0.5
                step = step * 0.5
                lod = lod - 1
                pos = pos - stepDir
                d = d - step
            else:
                if lod < 2:
                    c = textureLod(uShadowVolume, pos, lod + 1).x
                    if c == 0:
                        lod = lod + 1
                        stepDir = stepDir * 2.0
                        step = step * 2.0
                        pos = pos + stepDir
                        d = d + step
                # If lod == 2 and c == 0, do nothing, loop continues
    return dist
```

## Subtopics

### Hierarchical Shadow Volume Raycast (Non-Sparse)
A hierarchical raycasting algorithm for shadow volumes that uses fixed step sizes and LODs, traversing from base distance to further distances using specific LOD levels (0, 1, 2) [S1, unofficial]. It differs from the sparse variant by not dynamically adjusting step sizes based on occupancy bits, instead using predefined distance thresholds (`baseDistance`) to switch between LOD levels [S1, unofficial]. It is chosen when the shadow volume is not sparse or when a simpler, fixed-step traversal is preferred [S1, unofficial].

### Directional Shadow Raycast
Performs a directional shadow raycast against the sparse shadow volume, including screen-space blocking check and position jittering [S1, unofficial]. It first checks if the ray is blocked in screen space; if so, it returns 0.0. Otherwise, it jitters the position and calls `raycastShadowVolumeSparse` with the jittered position and direction [S1, unofficial]. It is used to reduce aliasing and improve shadow quality [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Origin: The method is described in the context of a voxel rendering project, specifically in the `main_sv.cpp` file of the TTFH/Voxel-Render repository [S1, unofficial]. It is part of a broader set of voxel grid raycasting techniques [S1, unofficial].

## Sources
[S1] TTFH/Voxel-Render, https://github.com/TTFH/Voxel-Render, fetched 2026-10-27 (unofficial)
