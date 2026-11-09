# Temporal Differential Rendering

Also called: Dirty-brick partial re-render

## Overview
Temporal Differential Rendering accelerates volume rendering by skipping the computation of screen tiles that have not changed since the previous frame, relying on the assumption that camera motion and volume updates are small between frames [S1, S2, unofficial]. It sets itself apart from sibling methods like [Reprojection with Temporal Accumulation](8337-reprojection-with-temporal-accumulation.md) by explicitly tracking "dirty" data structures (bricks) on the CPU to cull entire 8×8 pixel tiles on the GPU, rather than relying solely on pixel-level reprojection or accumulation buffers [S1, unofficial].

## How it works
The method operates in two main phases: a CPU-side dirty tile projection and a GPU-side compute shader execution.

**Inputs and Outputs:**
*   **Input:** A list of "dirty bricks" (volume data blocks modified by physics or user edits) [S1, unofficial]. Previous frame's first-hit distance texture [S1, unofficial].
*   **Output:** Final color buffer for changed tiles; unchanged tiles are skipped [S1, unofficial].

**Data Structures:**
*   **Dirty Bricks List:** A list of brick identifiers modified since the last frame [S1, unofficial].
*   **First-Hit Texture:** An `r32float` texture storing the distance to the first opaque surface hit for each 8×8 tile [S1, unofficial].

**Steps:**
1.  **Dirty Tile Projection (CPU):** The CPU iterates through the list of dirty bricks. For each dirty brick, it projects the brick's 3D bounding box into screen space to determine which 8×8 pixel tiles overlap with it [S1, unofficial]. These tiles are marked as "dirty" [S1, unofficial]. This logic is implemented in `src/temporal.rs` [S1, unofficial].
2.  **Change Detection (CPU):** If no dirty bricks are present, both compute passes are skipped entirely for the frame [S1, unofficial].
3.  **Pre-pass (GPU):** For dirty tiles, a compute shader executes a ray march at 1/8 resolution (one ray per 8×8 pixel block) [S1, unofficial]. This ray walks the world at tile granularity and writes the first-hit distance to the `r32float` texture [S1, unofficial].
4.  **Main Pass (GPU):** The main compute shader processes the dirty tiles [S1, unofficial]. It fast-forwards each ray to the depth stored in the `r32float` texture minus a 16-voxel safety margin [S1, unofficial]. This skips the traversal of empty space up to the first hit [S1, unofficial].
5.  **Shading:** Rays continue marching from the fast-forwarded depth to render the volume [S1, unofficial].

**Parameters:**
*   **Tile Size:** 8×8 pixels [S1, unofficial].
*   **Safety Margin:** 16 voxels [S1, unofficial].
*   **Pre-pass Resolution:** 1/8 of the screen resolution [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The safety margin (16 voxels) must be large enough to account for camera movement and potential errors in the first-hit distance, but small enough to avoid missing visible geometry [S1, unofficial].
*   **Boundary:** If a dirty brick is partially outside the view frustum, the projection logic must correctly handle culling [S1, unofficial].

## Subtopics

### Partial Volume Rendering Update
This variant integrates partial rendering into the visibility detection loop. Between passes of the visibility detection algorithm, a partial volume rendering pass is performed to complete the image [S2, unofficial]. The updated image is used to refresh occluding data, ensuring that subsequent visibility tests account for newly loaded or rendered bricks [S2, unofficial]. The frequency of total scene rendering must be chosen carefully as it is expensive [S2, unofficial]. A potential future extension involves progressive rendering by storing ray state (position, direction, hierarchy position, DDA stack state) in image-space buffers alongside color and opacity, allowing ray-tracing resumption between passes [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of dirty tiles, not the total screen resolution [S1, unofficial].
*   **Memory Complexity:** Requires storage for the first-hit distance texture (`r32float` per tile) [S1, unofficial].
*   **Performance:** Significant speedup when scene changes are localized [S1, unofficial].
*   **Strengths:** Efficiently handles dynamic volumes where only small regions change [S1, unofficial].
*   **Limitations:** Requires CPU-GPU synchronization to track dirty bricks [S1, unofficial]. Not suitable for fully dynamic scenes with global changes [S1, unofficial].

## Background
The method is described in the context of the `voxelG` project, which implements a voxel-based renderer with temporal coherence optimizations [S1, unofficial]. The concept of partial rendering updates for visibility detection is discussed in research on interactive exploration of procedural volumes [S2, unofficial].

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
