# Temporal Differential Rendering

Also called: Dirty-brick partial re-render

## Overview
Temporal Differential Rendering accelerates volume rendering by skipping the processing of screen tiles that correspond to unchanged regions of the volume, referred to as "dirty bricks" [S1, unofficial]. It solves the problem of redundant computation in dynamic scenes where only small portions of the volume change between frames [S1, unofficial]. Unlike [Reprojection with Temporal Accumulation](8337-reprojection-with-temporal-accumulation.md), which blends pixel values, this method operates at the tile level to bypass ray casting entirely for static areas [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A list of "dirty bricks" (voxel regions modified by physics or user edits) [S1, unofficial].
- Input: Previous frame's first-hit distance buffer (optional, for fast-forwarding) [S1, unofficial].
- Output: Final color buffer [S1, unofficial].

**Data Structures:**
- **Dirty Brick List:** A CPU-side list of brick identifiers that have changed since the last frame [S1, unofficial].
- **Tile Map:** An 8×8 pixel screen tile grid [S1, unofficial].
- **First-Hit Distance Texture:** An `r32float` texture storing the distance to the first opaque surface hit per tile [S1, unofficial].

**Steps:**
1. **Dirty Brick Projection (CPU):** Project the bounding volumes of all dirty bricks into screen space to identify which 8×8 tiles they affect [S1, unofficial]. Mark these tiles as "dirty" [S1, unofficial].
2. **Tile Status Check (Compute Shader):** For each tile, check if it is marked dirty [S1, unofficial].
    - If the tile is **clean** (no dirty bricks projected into it), the compute shader performs an early-out, skipping ray casting for this tile [S1, unofficial].
    - If the tile is **dirty**, proceed to step 3 [S1, unofficial].
3. **Ray Casting (Compute Shader):**
    - **Pre-pass (Optional):** At 1/8 resolution, one ray per 8×8 pixel block walks the world at tile granularity and writes the first-hit distance to the `r32float` texture [S1, unofficial].
    - **Main Pass:** Fast-forward each ray to the depth indicated by the first-hit distance minus a 16-voxel safety margin [S1, unofficial]. Perform ray casting and shading for the remaining path [S1, unofficial].
4. **Skip Condition:** If no dirty bricks are detected in step 1, both the pre-pass and main pass compute shaders are skipped entirely for the frame [S1, unofficial].

**Parameters:**
- **Tile Size:** 8×8 pixels [S1, unofficial].
- **Safety Margin:** 16 voxels (used when fast-forwarding rays) [S1, unofficial].
- **Distance LOD Threshold:** 400 voxels (beyond this, DDA terminates at brick granularity) [S1, unofficial].
- **Render Scale:** `RENDER_SCALE` (default 1:1, bilinear upscale applied in blit) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** The safety margin (16 voxels) must be large enough to account for camera movement or volume expansion between the pre-pass and main pass to avoid popping artifacts [S1, unofficial].
- **Boundaries:** Dirty bricks near tile boundaries must correctly mark all overlapping tiles [S1, unofficial].

## Subtopics

### Partial Volume Rendering Update
A variant where a partial volume rendering pass is performed between passes of a visibility detection algorithm [S2, unofficial].
- **Mechanism:** After a visibility pass, a partial render completes the image and updates occluding data, ensuring subsequent visibility tests account for newly loaded bricks [S2, unofficial].
- **Parameter:** The frequency of total scene rendering must be chosen carefully as it is expensive [S2, unofficial].
- **Pitfall/Future Work:** Progressive rendering via ray-tracing resumption could store ray state (position, direction, hierarchy position, DDA stack state) in image-space buffers alongside color and opacity [S2, unofficial].

## Operating characteristics
- **Time Complexity:** Proportional to the number of dirty tiles, not the total screen resolution [S1, unofficial].
- **Memory Complexity:** Requires storage for the dirty brick list and the first-hit distance texture (`r32float`) [S1, unofficial].
- **Performance:** Significant speedup in scenes with low temporal change [S1, unofficial].
- **Strengths:** Eliminates redundant ray casting for static regions [S1, unofficial].
- **Limitations:** Requires CPU-GPU synchronization for dirty brick projection [S1, unofficial].

## Background
Originated in the voxelG project by mstampfli [S1, unofficial]. The concept of partial rendering updates for visibility detection was also explored in the context of procedural volume exploration [S2, unofficial].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
