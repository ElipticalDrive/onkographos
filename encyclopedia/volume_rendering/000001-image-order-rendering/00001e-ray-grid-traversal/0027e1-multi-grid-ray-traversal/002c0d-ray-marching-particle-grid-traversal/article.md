# Ray Marching Particle Grid Traversal
Also called: Hybrid Ray Marching Particle Grid Traversal, Low-Res Culling High-Res Hit Test

## Overview
A hybrid ray marching technique that traverses a voxel grid to find particle hits, using a low-resolution grid for culling and a high-resolution grid for precise hit testing [S1, unofficial]. It sets itself apart from standard ray marching by introducing a two-tier grid hierarchy to optimize the search for sparse particle distributions, avoiding expensive hit tests in empty regions [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin ($pos_0$), ray direction ($rayDirection$), high-resolution particle position buffer ($particlesPositionBuffer$), high-resolution particle index buffer ($particlesIndicesBuffer$), low-resolution occupancy grid buffer ($gridBuffer$).
*   **Output:** Hit color, hit position, hit velocity, and a boolean indicating success.

**Data Structures:**
*   $particlesIndicesBuffer$: Array of `i32`. Stores particle IDs. Layout is linearized 3D voxel indices: $index1D = mapPos.x + res \times mapPos.y + res^2 \times mapPos.z$. Each high-res voxel can store up to 8 particle IDs, accessed via $8 \times index1D + z$ where $z \in [0, 7]$.
*   $particlesPositionBuffer$: Array of `vec4f`. Stores particle positions (RGB) and potentially other attributes.
*   $gridBuffer$: Array of `i32`. Stores occupancy flags (1 if occupied, 0 if empty) for the low-resolution grid.

**Parameters:**
*   $MAX\_ITERATIONS$: 400 [S1, unofficial].
*   $GRID\_RATIO$: 8 [S1, unofficial].
*   $RADIUS$: 0.19 (squared radius for hit check) [S1, unofficial].
*   $EPSILON$: $1.e-10$ [S1, unofficial].
*   $acne$: 0.05 (offset to prevent texture sampling artifacts) [S1, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Calculate initial position in map space: $mapPos = floor(pos_0)$.
    *   Calculate side distances for slab crossing:
        $$sideDist = (rayStep \times (mapPos - pos_0 + 0.5) + 0.5) \times deltaDist$$
        [S1, unofficial]
    *   Set $highResolution = true$.
    *   Set $d = \infty$ (distance to closest hit).
    *   Set $index = -1000$ (hit particle ID).

2.  **Ray Marching Loop:**
    For $i$ from 0 to $MAX\_ITERATIONS$:
    *   **Step Direction:**
        $$stepForward = step(sideDist.xyz, sideDist.yxy) \times step(sideDist.xyz, sideDist.zzx)$$
        [S1, unofficial]
        *(Note: `step(a, b)` returns 1.0 if $a < b$, else 0.0. This logic determines the next axis to cross.)*
    *   **Update State:**
        $$sideDist += stepForward \times deltaDist$$
        $$pos += stepForward \times rayStep$$
        $$mapPos = floor(pos)$$
        [S1, unofficial]
    *   **Distance to Next Boundary:**
        $$t = min(sideDist.x, min(sideDist.y, sideDist.z))$$
        [S1, unofficial]
    *   **Border Check:**
        Check if the ray is near the volume boundaries:
        $$borders = length(pos.yz / uniforms.resolution - vec2(0.5)) > INNER\_RADIUS$$
        Perform similar checks for x-boundaries. If borders are true, `continue` (skip hit testing) [S1, unofficial].
    *   **Low-Res Grid Index Calculation:**
        If $highResolution$:
        $$lowGrid = mapPos / GRID\_RATIO$$
        [S1, unofficial]
        Else:
        $$lowGrid = mapPos$$
        [S1, unofficial]
        $$indexGrid = lowGrid.x + lowRes \times lowGrid.y + lowRes^2 \times lowGrid.z$$
        [S1, unofficial]
    *   **Occupancy Check:**
        $$resolution = gridBuffer[indexGrid] == 1$$
        [S1, unofficial]
    *   **Mode Switching (High to Low):**
        If $highResolution$ and not $resolution$:
        *   Switch to low-res mode.
        *   Update position: $pos0\_l = (pos0\_h + t \times rayDirection) / GRID\_RATIO$
        *   $pos = pos0\_l$
        *   $mapPos = floor(pos)$
        *   Recalculate $sideDist$ [S1, unofficial].
    *   **Mode Switching (Low to High):**
        If not $highResolution$ and $resolution$ is true:
        *   Switch back to high-res mode.
        *   Update position: $pos0\_h = GRID\_RATIO \times (pos0\_l + 0.9999 \times t \times rayDirection)$
        *   $pos = pos0\_h$
        *   $mapPos = floor(pos)$
        *   Recalculate $sideDist$ [S1, unofficial].
    *   **High-Res Hit Testing:**
        If $highResolution$ and $resolution$:
        *   Calculate 1D index in high-res grid:
            $$index1D = mapPos.x + res \times mapPos.y + res^2 \times mapPos.z$$
            [S1, unofficial]
        *   Loop $z$ from 0 to 7:
            *   $particleId = 8 \times index1D + z$
            *   $voxelData = particlesIndicesBuffer[particleId]$
            *   If $voxelData == 0$, break loop [S1, unofficial].
            *   $center = particlesPositionBuffer[voxelData].rgb$
            *   Check if center is within valid bounds: $center.x > 0.1 \times uniforms.resolution$ [S1, unofficial].
            *   Check hit: If $checkHit(initPos, center, rayDirection)$ and $l < d$:
                *   $index = voxelData$
                *   $d = l$
                [S1, unofficial]
    *   **Hit Return:**
        If $index > -1000$:
        *   Set $hitColor$, $hitPosition$, $hitVelocity$ from buffers.
        *   Return true [S1, unofficial].
    *   **Update Flag:**
        $$highResolution = resolution$$
        [S1, unofficial]

3.  **Fallback:**
    If loop completes without hit, return false [S1, unofficial].

**Pitfalls:**
*   The `checkHit` function uses a squared radius ($RADIUS$) to avoid square root calculations. The distance check must be squared as well ($d = l^2 - t_{CA}^2$) [S1, unofficial].
*   Precision issues may arise when switching between high-res and low-res modes due to floating-point rounding in position updates [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of ray steps, but significantly reduced in practice due to low-res culling.
*   **Memory Complexity:** $O(V_{high} + V_{low})$ for storing particle data and occupancy grid.
*   **Strengths:** Efficiently handles sparse particle distributions by skipping empty high-res voxels.
*   **Limitations:** Requires maintaining two grid resolutions and switching logic. Performance degrades if the low-res grid is too coarse or if particles are densely packed.
*   **When to use:** Sparse particle systems where most voxels are empty.
*   **When not to use:** Dense volumetric data where high-res traversal is necessary everywhere.

## Background
Originated in the context of WebGPU-based voxel cone tracing and particle rendering, specifically in the `WebGPUVoxelConeTracing` repository by HectorArellanoDev [S1, unofficial]. It is an adaptation of multi-grid traversal techniques to optimize particle hit testing in real-time rendering.

## Sources
[S1] HectorArellanoDev/WebGPUVoxelConeTracing, https://github.com/HectorArellanoDev/WebGPUVoxelConeTracing, fetched 2026-10-26 (unofficial)
