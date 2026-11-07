# Temporal Differential Rendering

Also called: Dirty-brick partial re-render

## Overview
Temporal Differential Rendering accelerates volume rendering by skipping the computation of screen tiles that have not changed since the previous frame. It solves the problem of redundant ray-marching costs in dynamic scenes where only small portions of the volume data (e.g., due to physics or user edits) are modified. It sets itself apart from [Reprojection with Temporal Accumulation](8337-reprojection-with-temporal-accumulation.md) by relying on explicit state tracking of "dirty" regions rather than blending previous frame buffers, and from [Exploiting Temporal Coherence (C-Buffer)](5560-exploiting-temporal-coherence-c-buffer.md) by operating at the tile level with early-out logic rather than full-frame caching.

## How it works
**Inputs and Outputs:**
*   **Input:** The set of "dirty" bricks (volume data regions) that have changed since the last frame, the previous frame's output, and the current camera/view parameters.
*   **Output:** The current frame's color buffer.

**Data Structures:**
*   **Dirty Brick List:** A list of brick identifiers that require re-evaluation.
*   **Tile Status Buffer:** A buffer mapping 8×8 pixel screen tiles to a "clean" or "dirty" state.
*   **Hit Distance Texture:** An `r32float` texture storing the first-hit distance for each 8×8 tile, used for depth culling.

**Steps:**
1.  **Identify Dirty Bricks:** Determine which bricks in the volume hierarchy have changed due to physics simulation or user edits.
2.  **Project Dirty Bricks to Tiles:** On the CPU, project the bounding boxes of the dirty bricks onto the screen to identify which 8×8 pixel tiles are affected. Mark these tiles as "dirty" in the Tile Status Buffer. All other tiles are marked "clean" [S1, unofficial].
3.  **Pre-pass (Tile Granularity):** For dirty tiles, execute a pre-pass at 1/8 resolution (one ray per 8×8 block). This ray walks the volume hierarchy and writes the first-hit distance to the Hit Distance Texture. Clean tiles skip this pass [S1, unofficial].
4.  **Main Pass (Ray Marching):**
    *   For each pixel in a **dirty** tile: Fast-forward the ray to the depth specified by the Hit Distance Texture minus a 16-voxel safety margin. Perform full ray marching and shading from that point [S1, unofficial].
    *   For each pixel in a **clean** tile: The compute shader early-outs, reusing the previous frame's result for that tile [S1, unofficial].
5.  **Skip Check:** If no dirty bricks are detected, both the pre-pass and main pass are skipped entirely for the frame [S1, unofficial].

**Parameters:**
*   **Tile Size:** 8×8 pixels [S1, unofficial].
*   **Safety Margin:** 16 voxels (subtracted from the first-hit distance to prevent z-fighting or missed intersections due to precision) [S1, unofficial].
*   **Resolution Scale:** The internal rendering resolution can be scaled (e.g., `RENDER_SCALE`), with bilinear upscaling applied during the final blit. Ships at native 1:1 [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** The 16-voxel safety margin is critical to handle floating-point inaccuracies in the first-hit distance.
*   **Boundary:** If a dirty brick is small and does not cover a full 8×8 tile, the entire tile is re-rendered, potentially wasting computation on unchanged pixels within that tile.
*   **Distance LOD:** Beyond 400 voxels, the Distance Discrete Approximation (DDA) terminates at brick granularity, shading with the brick's topmost solid material to maintain surface color for distant terrain [S1, unofficial].

## Subtopics

### Partial Volume Rendering Update
A variant where partial rendering is interleaved with visibility detection. Between passes of the visibility algorithm, a partial volume rendering pass completes the image to refresh occluding data, ensuring visibility tests account for newly loaded bricks. The frequency of total scene rendering must be chosen carefully as it is expensive. A future consideration is progressive rendering via ray-tracing resumption, storing ray state (position, direction, hierarchy position, DDA stack state) in image-space buffers alongside color and opacity [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Proportional to the number of dirty tiles. If the scene is static, complexity drops to $O(1)$ (skip).
*   **Memory Complexity:** Requires storage for the Hit Distance Texture (`r32float` per tile) and Tile Status Buffer.
*   **Strengths:** Significant performance gains in dynamic scenes with localized changes. Avoids the ghosting artifacts associated with temporal accumulation.
*   **Limitations:** Inefficient for global changes (e.g., camera rotation, global lighting changes) where most tiles become dirty. Tile-based granularity (8×8) may lead to over-rendering for small changes.
*   **When to use:** Dynamic voxel simulations (fluids, destruction) where changes are localized.
*   **When not to use:** Static scenes (use caching) or scenes with frequent global updates.

## Background
The method is implemented in the voxelG project, a real-time voxel renderer. It leverages the hierarchical nature of voxel data (bricks) and the coherence of screen-space tiles to optimize ray marching. The approach is distinct from traditional temporal reprojection as it relies on explicit dirty-state tracking rather than motion vectors.

## Sources
[S1] mstampfli/voxelG, https://github.com/mstampfli/voxelG, fetched 2026-10-24 (unofficial)
[S2] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
