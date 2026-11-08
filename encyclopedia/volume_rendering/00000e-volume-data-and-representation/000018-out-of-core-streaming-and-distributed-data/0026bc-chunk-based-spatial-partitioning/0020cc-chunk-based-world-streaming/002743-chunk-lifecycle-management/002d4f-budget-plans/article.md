# Budget Plans

Also called: chunk budget policies

## Overview
Budget Plans manage chunk loading policies for chunked volumes, replacing the single octree-focused policy with presets for exploration, whole-volume figures, and frame-rate holding [S1, unofficial]. Unlike [Dynamic Chunk Generation with Radius](4002-dynamic-chunk-generation-with-radius.md) which focuses on geometric radius, or [Chunk Loading/Unloading Scanner](4404-chunk-loading-unloading-scanner.md) which manages eviction, Budget Plans define the target resident count and detail level based on user intent [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A budget plan configuration object or preset name.
*   Output: A resolved `BudgetPlan` object containing `minLevel`, `halo`, `radius`, and `gridDims`.

**Data Structures:**
The `BudgetPlan` type contains:
*   `minLevel`: Integer, the minimum LOD level to stream.
*   `halo`: Integer, the number of chunks surrounding the view volume to keep resident.
*   `radius`: String or number, defines the spatial extent for detail shrink passes.
*   `gridDims`: Array of 3 integers `[x, y, z]`, defining the exact lattice block counts per axis (used in Exact Lattice Grid variant).

**Steps:**
1.  **Resolve Plan:** Call `resolveBudgetPlan` with the desired preset or explicit options. The resolution follows a precedence order (lowest to highest):
    1.  Focus baseline (default octree policy).
    2.  Named preset (`focus`, `uniform`, `interactive`).
    3.  Explicit plan object.
    4.  Individual options.
    This ensures existing calls retain their behavior [S1, unofficial].
2.  **Apply Mid-Session:** Call `setBudgetPlan` to switch plans during runtime. This triggers a re-fold over load options [S1, unofficial].
3.  **State Preservation:** During the switch, `minLevel` and `halo` are derived from the current state. This prevents the switch from undoing `setMaxDetail` calls or undercutting wider kernels already streamed [S1, unofficial].
4.  **Uniform Detail Shrink:** If `radius` is set to `'volume'`, a uniform detail shrink pass is enabled. This pass only affects detail levels beyond the specified radius, allowing the budget pass to converge on the finest uniform level that fits within the budget [S1, unofficial].
5.  **Lattice Indexing (Exact Lattice Grid):** If `gridDims` is specified, the system replaces octree-based brick sizing with an exact lattice. `chunkAtVoxel` indexes this lattice using the real `gridDims` and `stride` set on the plan [S1, unofficial].

**Parameters:**
*   **Presets:** Three presets are available: `focus`, `uniform`, and `interactive` [S1, unofficial].
*   **targetFrameMs:** A parameter that is resolved and readable, but currently no action is taken on it [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Chunk 0 Eviction:** If chunk 0 is evicted, the volume may blank out (WebGL2) because `draw()` early-returns on null `volumeTexture`/`volumeGradientTexture`. These textures are best-effort aliases of chunk 0; real textures bind per chunk. A refocus that evicts chunk 0 silently kills the volume [S1, unofficial].
*   **Fallback Sampling:** If the primary sampler fails, the system falls back to `vol.pickSampler` (mm-space, backed by the coarse floor) to ensure both backends function [S1, unofficial].

## Subtopics

### Exact Lattice Grid
Replaces octree-based brick sizing with an exact lattice defined by `gridDims`, allowing specific block counts previously unreachable due to scalar `cellEdge` sizing [S1, unofficial].
*   **How it works:** Set real `gridDims`/stride on the plan so `chunkAtVoxel` indexes it. An axis grows when a brick would exceed `deviceLimit`, as `emitBrick` clamps oversize textures which squashes sampling; callers read the built plan's own `gridDims` back [S1, unofficial].
*   **Pitfall:** Previous `chunkVolumeMultiLOD` sized bricks from scalar `cellEdge`, causing counts to step 2, 8, 12, 16, 27, 4 on `hoa_heart` L5 (181x186x246); 4 or 9 were unreachable. `maxBricks` only coarsened by raising the level floor, fighting level control [S1, unofficial].

## Operating characteristics
*   **Complexity:** Not given in the sources.
*   **Performance:** The `targetFrameMs` parameter is resolved but not yet acted upon, so frame-rate holding performance is not currently optimized by this system [S1, unofficial].
*   **Strengths:** Allows mid-session switching without undoing manual detail settings; supports exact block counts for uniform volumes.
*   **Limitations:** Chunk 0 eviction can crash the volume if not handled; `targetFrameMs` is inert.

## Background
Originated in the Niivue project to address the limitation of having only one octree-focused policy, which was suitable for exploration but poor for whole-volume figures or frame-rate holding [S1, unofficial]. The design doc is located at `docs/budget-plans.md`, with remaining streaming work tracked in `docs/streaming-todos.md` [S1, unofficial].

## Sources
[S1] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
