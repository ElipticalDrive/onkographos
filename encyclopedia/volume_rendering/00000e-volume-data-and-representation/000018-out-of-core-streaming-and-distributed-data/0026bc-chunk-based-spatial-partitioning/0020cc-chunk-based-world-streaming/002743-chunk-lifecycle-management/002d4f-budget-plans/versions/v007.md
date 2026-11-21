# Budget Plans

Also called: chunk budget policies

## Overview
Budget Plans manage chunk loading policies for chunked volume rendering, replacing the single octree-focused policy with presets for exploration, whole-volume figures, and frame-rate holding [S1, unofficial]. It solves the problem of rigid streaming behavior by allowing mid-session switching and re-folding of load options based on user intent [S1, unofficial]. Unlike siblings that focus on geometric generation or scanner mechanics, Budget Plans shape the octree structure and residency targets to balance detail and performance [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A configuration object specifying the desired plan (preset or explicit), current scene state (minLevel, halo), and optional explicit options.
- Output: A resolved `BudgetPlan` object containing `gridDims`, `stride`, `radius`, and other parameters used by the streaming engine.

**Data Structures:**
- `BudgetPlan`: An object containing:
  - `gridDims`: Array of integers `[x, y, z]` representing the number of bricks per axis.
  - `stride`: Array of integers `[x, y, z]` representing the stride for chunk indexing.
  - `radius`: Number or string ('volume') defining the detail shrink pass radius.
  - `minLevel`: Integer minimum LOD level.
  - `halo`: Integer halo size.

**Steps:**
1. **Resolve Plan:** Call `resolveBudgetPlan` with the input configuration.
   - Determine the effective plan based on precedence: focus baseline < named preset < explicit plan object < individual options [S1, unofficial].
   - If a named preset is used, apply its default parameters.
2. **Apply Radius:** If `radius` is set to 'volume', enable a uniform detail shrink pass. This pass only affects levels beyond the specified radius and converges on the finest uniform level that fits within the budget [S1, unofficial].
3. **Switch Plan:** If `setBudgetPlan` is called mid-session:
   - Re-fold over the current load options.
   - Preserve `minLevel` and `halo` from the current state to prevent undoing `setMaxDetail` or undercutting wider kernels already streamed [S1, unofficial].
4. **Handle Lattice Grid (if applicable):** If `gridDims` is specified:
   - Set `gridDims[a]` bricks per axis, all drawn from `minLevel`.
   - Set `stride` on the plan so `chunkAtVoxel` can index it correctly.
   - For each axis, if a brick would exceed `deviceLimit`, grow the axis count (since `emitBrick` clamps oversize textures, which squashes sampling) [S1, unofficial].
   - Return the built plan's `gridDims` to the caller.

**Parameters:**
- **Presets:** Three presets are available: `focus`, `uniform`, `interactive` [S1, unofficial].
- **Precedence:** Lowest to highest: focus baseline, named preset, explicit plan object, individual options [S1, unofficial].
- **targetFrameMs:** Resolved and readable, but currently not acted upon [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Chunk 0 Eviction:** If chunk 0 is evicted, the volume may appear blanked because `draw()` early-returns on null `volumeTexture` / `volumeGradientTexture`, which are best-effort aliases of chunk 0 [S1, unofficial].
- **Unreachable Counts:** Previous `chunkVolumeMultiLOD` sized bricks from scalar `cellEdge`, causing counts to step 2, 8, 12, 16, 27, 4 on `hoa_heart` L5 (181x186x246), making 4 or 9 unreachable [S1, unofficial]. `maxBricks` only coarsened by raising level floor, fighting level control [S1, unofficial].
- **Degenerate Plans:** Multi-LOD plans previously returned degenerate `[1,1,1]` for `gridDims` [S1, unofficial].

## Subtopics

### [Block Loading Badges](6011-block-loading-badges.md)
Block Loading Badges provide per-pane visibility into the status of GPU upload queues for chunk-based volume rendering. They solve the problem of opaque streaming progress by tracking the resident count movement of fetched blocks, distinguishing actual data acquisition from planned requests. Unlike general budget plans that shape octree structure, badges specifically monitor the GPU residency cap and upload queue depth, ensuring the UI reflects only the blocks the stream actually requested and fetched.

### [Capacity Handling](14405-capacity-handling.md)
Capacity Handling converts continuous block quotas derived from logarithmic smoothing into integer budgets that strictly respect block capacity limits and sum to the global target count $K$. It serves as the final step in Budget Plans, ensuring that the number of retained Gaussians per chunk is a valid integer before Quadratic Selection determines which specific Gaussians to keep. It differs from simple rounding by handling capacity clipping and deficit redistribution through iterative passes.

### Exact Lattice Grid
Exact Lattice Grid replaces octree-based brick sizing with an exact lattice defined by `gridDims`, allowing specific block counts that were previously unreachable due to scalar `cellEdge` sizing [S1, unofficial]. It sets real `gridDims`/`stride` on the plan so `chunkAtVoxel` indexes it. An axis grows when a brick would exceed `deviceLimit`, since `emitBrick` clamps an oversize texture and that squashes the sampling, so callers read the built plan's own `gridDims` back [S1, unofficial]. Previous `chunkVolumeMultiLOD` sized bricks from scalar `cellEdge`, so counts stepped 2, 8, 12, 16, 27, 4 on `hoa_heart` L5; 4 or 9 were unreachable. `maxBricks` only coarsened by raising level floor, fighting level control [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Allows mid-session switching of streaming policies; supports uniform detail shrink for whole-volume figures; provides exact lattice control for specific block counts.
- **Limitations:** `targetFrameMs` is not yet acted upon; Chunk 0 eviction can blank the volume; previous brick sizing led to unreachable block counts.
- **When to Use:** Use for exploring (focus), whole-volume figures (uniform), or holding frame rate while rotating (interactive). Use Exact Lattice Grid when specific block counts are required.
- **When Not to Use:** Not suitable for scenarios requiring dynamic frame-rate adaptation (as `targetFrameMs` is inert).

## Background
Origin: Introduced in [S1, unofficial] as part of the `niivue` library's streaming improvements. The design doc is located at `docs/budget-plans.md`. Remaining streaming work is tracked in `docs/streaming-todos.md` (off-thread chunk decode is the next item) [S1, unofficial].

## Sources
[S1] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
