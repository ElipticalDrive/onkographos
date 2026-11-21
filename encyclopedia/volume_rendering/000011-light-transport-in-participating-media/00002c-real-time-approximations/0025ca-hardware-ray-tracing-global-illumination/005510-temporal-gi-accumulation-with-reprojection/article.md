# Temporal GI Accumulation with Reprojection

Also called: RT variant GI, Reprojected GI

## Overview
This method accumulates hardware ray-traced indirect illumination over time by reprojecting the previous frame's GI buffer into the current frame, reducing noise and convergence time for dynamic scenes. It differs from [Hardware Ray Tracing Global Illumination](3648-hardware-ray-tracing-global-illumination.md) by adding temporal history blending and from [Dynamic Diffuse Global Illumination (DDGI)](8142-dynamic-diffuse-global-illumination-ddgi.md) by relying on per-pixel reprojection rather than sparse probe caching for temporal stability.

## How it works
**Inputs:**
- Current frame camera state: `world_origin`, `prev_origin`, `prev_forward`, `prev_right`, `prev_up`, `tan_half_fov`, `resolution` (x, y).
- Previous frame G-buffer position buffer: `light_in` (texture_2d<f32>).
- Previous frame GI buffer: `gi_in` (texture_2d<f32>).
- Surface shading point `p` (world space), normal `n`, view vector `v`.
- Ray tracing gather function `rt_gather_indirect(p, n, seed)` returning a radiance value.
- Probe sampling function `sample_probes(p, n, v)` returning a radiance value.
- Parameters: `GI_ENABLE`, `GI_MAX_T`, `GI_STRENGTH`, `GI_PROBE_MODE`, `REPROJ_EPS2`.

**Outputs:**
- `gi_out`: `texture_storage_2d<rgba32float, write>` storing the final accumulated indirect color as `vec4<f32>(acc, 1.0)`.

**Data Structures:**
- `gi_in`: 2D texture of `f32` RGBA values. Stores the temporally accumulated indirect illumination from the previous frame.
- `light_in`: 2D texture of `f32` RGBA values. Stores the world-space position of the surface point corresponding to the pixel.
- `gi_out`: 2D storage texture of `f32` RGBA values. Target for the current frame's result.

**Steps:**

1.  **Early Exit Check:** If `GI_ENABLE < 0.5` or the ray hit distance `t >= GI_MAX_T`, store `vec4<f32>(0.0, 0.0, 0.0, 1.0)` to `gi_out` and return `0.0`.
2.  **Fade Factor Calculation:** Calculate a distance-based fade factor to prevent artifacts at the maximum ray distance:
    $$ fade = 1.0 - \text{smoothstep}(GI\_MAX\_T \times 0.6, GI\_MAX\_T, t) $$
3.  **Indirect Illumination Sample:**
    -   If `GI_PROBE_MODE > 0.5`:
        -   Sample irradiance from probes: $gi = \text{sample\_probes}(p, n, v) \times (GI\_STRENGTH \times fade)$.
        -   Store `vec4<f32>(gi, 1.0)` to `gi_out` and return `gi`.
    -   Otherwise (per-pixel ray tracing):
        -   Compute fresh sample: $acc = \text{rt\_gather\_indirect}(p, n, \text{seed}) \times fade$.
4.  **Temporal Reprojection Attempt:**
    -   If `camera.reproject_lighting > 0.5`:
        -   Calculate relative hit position: $hitpos\_rel = p - \text{camera.world\_origin}$.
        -   Calculate vector from previous camera origin: $d = p - \text{camera.prev\_origin}$.
        -   Calculate depth along previous forward vector: $pz = \text{dot}(d, \text{camera.prev\_forward})$.
        -   If $pz > 0.01$:
            -   Project to Normalized Device Coordinates (NDC):
                $$ ndc = \left( \frac{\text{dot}(d, \text{camera.prev\_right})}{pz \times \text{camera.tan\_half\_fov} \times \text{aspect}}, \frac{\text{dot}(d, \text{camera.prev\_up})}{pz \times \text{camera.tan\_half\_fov}} \right) $$
                where $\text{aspect} = \text{camera.resolution.x} / \text{camera.resolution.y}$.
            -   Convert NDC to UV coordinates:
                $$ uvp = (ndc.x \times 0.5 + 0.5, 0.5 - ndc.y \times 0.5) $$
            -   If $uvp \in [0, 1)$:
                -   Load previous G-buffer position: $g = \text{textureLoad}(\text{light\_in}, \lfloor uvp \times \text{camera.resolution} \rfloor, 0)$.
                -   Check position match: $dpos = g.xyz - hitpos\_rel$.
                -   If $\text{dot}(dpos, dpos) < REPROJ\_EPS2$:
                    -   Load previous GI: $prev = \text{textureLoad}(\text{gi\_in}, \lfloor uvp \times \text{camera.resolution} \rfloor, 0).rgb$.
                    -   Blend with history: $acc = \text{mix}(acc, prev, 0.9)$.
5.  **Store Result:** Store $acc$ to `gi_out` as `vec4<f32>(acc, 1.0)` and return `acc`.

**Parameters:**
-   `GI_ENABLE`: Pipeline-override toggle. `1.0` (default) enables GI; `0.0` disables it. [S1, unofficial]
-   `GI_PROBE_MODE`: Toggle between probe cache (`1.0`) and legacy per-pixel bounce ray (`0.0`). [S1, unofficial]
-   `GI_MAX_T`: Maximum ray distance for GI. [S1, unofficial]
-   `GI_STRENGTH`: Multiplier for probe-based GI. [S1, unofficial]
-   `REPROJ_EPS2`: Squared epsilon for position matching during reprojection. Value not explicitly given in sources. [S1, unofficial]
-   **Blend Factor:** `0.9` (90% history, ~10-frame convergence). [S1, unofficial]

**Edge Cases and Pitfalls:**
-   **Reprojection Failure:** If the position match fails (e.g., due to occlusion changes, object motion, or depth discontinuities), the temporal history is discarded, and the fresh ray-traced sample is used. This can cause flickering if the reprojection is unstable.
-   **Zero Depth:** If $pz \le 0.01$, reprojection is skipped to avoid division by zero or invalid projections.
-   **Out of Bounds UV:** If the projected UV is outside $[0, 1)$, reprojection is skipped.
-   **Probe Mode:** In probe mode, temporal reprojection is bypassed, and results are returned directly from the probe sampler. [S1, unofficial]

## Operating characteristics
-   **Time Complexity:** $O(1)$ per pixel per frame, assuming constant-time texture lookups and ray tracing queries. The temporal reprojection adds a constant overhead of a few vector operations and texture loads.
-   **Memory Complexity:** $O(W \times H)$ for the `gi_in` and `light_in` buffers, where $W$ and $H$ are the screen resolution.
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** Performance scales linearly with resolution. The temporal aspect reduces the required number of ray samples per frame to achieve a noise-free result, effectively trading memory (for history buffers) for compute.
-   **Strengths:** Reduces noise significantly compared to single-frame ray tracing. Smooths out jitter from sparse sampling.
-   **Limitations:** Requires stable reprojection; fails on fast-moving objects or dynamic occlusions, leading to ghosting or flickering if not handled. Requires G-buffer position data.
-   **When to use:** Dynamic scenes with moderate camera movement where noise reduction is critical and hardware RT cores are available.
-   **When not to use:** Scenes with very fast-moving objects or frequent large-scale changes where reprojection fails frequently.

## Background
This method is implemented in the `voxelG` project by mstampfli. It is part of a real-time global illumination pipeline that uses hardware ray tracing cores for indirect lighting. The approach combines per-pixel ray tracing with temporal accumulation to improve visual quality.

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
