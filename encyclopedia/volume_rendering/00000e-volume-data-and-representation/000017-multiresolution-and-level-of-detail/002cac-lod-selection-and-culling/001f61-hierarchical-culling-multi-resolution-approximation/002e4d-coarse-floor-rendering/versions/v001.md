# Coarse Floor Rendering

Also called: whole-volume floor

## Overview
Coarse Floor Rendering prevents scene flicker during chunk plan swaps in hierarchical volume rendering by rendering the coarsest pyramid level as a continuous whole-volume floor behind the active octree [S1, unofficial]. Unlike previous implementations where unmatched bricks were synchronously destroyed and revealed the scene background, this method ensures no brick draws nothing while waiting for re-stream, providing continuous coverage [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The volume dataset with its multi-resolution pyramid structure; the current chunk plan defining which bricks are resident.
*   **Output:** A rendered frame where the background is occluded by the coarsest level's volume data, preventing visual gaps.

**Data Structures:**
*   **Coarsest Pyramid Level:** The lowest-resolution representation of the entire volume.
*   **Texture Upload:** The coarsest level must fit within a single texture upload constraint. If the coarsest level is too big for one texture upload, the floor is skipped [S1, unofficial].

**Parameters:**
*   `coarseFloor` (boolean): Enables the technique. Default is `true`. Set to `false` to opt out [S1, unofficial].
*   `chunkFadeMs` (number): Controls the transition duration for fine detail dissolving over the floor. Default is `120` ms. Set to `0` to disable fading [S1, unofficial].

**Steps:**
1.  **Fetch Coarsest Level:** Retrieve the coarsest pyramid level of the volume data once.
2.  **Install Floor:** Install this coarsest level as a whole-volume floor positioned behind the active octree structure [S1, unofficial].
3.  **Render as Volume:** Render the floor as a volume rather than a dark shell or solid background to maintain visual continuity [S1, unofficial].
4.  **Apply Fade Transition:** When new bricks stream in, apply a fade transition over the coarse floor to prevent popping artifacts. The fine detail dissolves in over the floor over the duration specified by `chunkFadeMs` [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Texture Size Limit:** If the coarsest level exceeds the maximum texture upload size, the floor is skipped, potentially reverting to flicker behavior if not handled [S1, unofficial].
*   **OME-Zarr Demo:** The OME-Zarr demo builds plans by hand, so the automatic floor never reached it; it must build one from the coarse data it already keeps resident [S1, unofficial].

## Subtopics

### Chunk Fade Transition
A transition effect applied on both backends to prevent popping artifacts when new bricks stream in, by fading fine detail over the coarse floor [S1, unofficial]. It uses the `chunkFadeMs` parameter (default 120 ms, 0 disables) to control the duration over which fine detail dissolves in over the coarse floor [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   Requires fetching and uploading the coarsest pyramid level once.
*   Adds a fade transition step during brick streaming.

**Measured Performance:**
*   Not given in the sources.

**Scaling:**
*   Skipped if the coarsest level is too big for one texture upload [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Eliminates flicker caused by brick destruction and re-streaming; provides continuous visual coverage [S1, unofficial].
*   **Limitations:** Dependent on texture upload size limits; requires manual adjustment for demos that build plans by hand (e.g., OME-Zarr) [S1, unofficial].

**When to Use:**
*   Use when chunk plan swaps cause visible flicker due to missing bricks.
*   Do not use if the coarsest level exceeds texture upload limits and no alternative floor generation is available [S1, unofficial].

## Background
Coarse Floor Rendering was introduced in the `niivue` library to address scene flicker during chunk plan swaps [S1, unofficial]. It evolved from a fix for swap flicker to cover every reason a brick could read wrong, including geometry, brightness, opacity, and reconstruction issues [S1, unofficial]. The technique was added to both backends to ensure consistent behavior [S1, unofficial].

## Sources
[S1] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
